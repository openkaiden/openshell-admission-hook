# openshell-adminssion-hook

A Rust **admission hook**. The
hook rewrites Windows bind-mount paths in sandbox creation requests to the path
the container machine sees (Docker Desktop / Podman machine).

```
openshell sandbox create --driver-config-json \
  '{"podman":{"mounts":[{"type":"bind","source":"C:\\Users\\me\\proj","target":"/sandbox/proj"}]}}'

            ── CreateSandbox ──▶ gateway ── modify_operation ──▶ admission hook (127.0.0.1:12777)
                                    ◀── JSON patch: source = /mnt/c/Users/me/proj ──
                                    ── podman driver creates the bind mount
```

## How it works

* **Admission hook.** A gRPC
  [gateway interceptor](https://docs.nvidia.com/openshell/latest/extensibility/gateway-interceptors)
  (`openshell.gateway_interceptor.v1.GatewayInterceptor`) bound to
  `openshell.v1.OpenShell/CreateSandbox` in the `modify_operation` phase.
  * It checks `spec.template.driverConfig.docker.mounts[]` and
    `spec.template.driverConfig.podman.mounts[]`.
  * For every `type: "bind"` entry whose `source` is a Windows path, it returns
    an RFC 6902 `replace` patch.
  * Windows paths it can't map (UNC shares, `..`, drive-relative `C:` / `C:foo`) are
    denied with `INVALID_ARGUMENT`.

  | Driver | `C:\Users\me\proj` becomes | Override |
  |---|---|---|
  | docker (Docker Desktop) | `/run/desktop/mnt/host/c/Users/me/proj` | `ADMISSION_HOOK_DOCKER_PREFIX` |
  | podman (Podman machine) | `/mnt/c/Users/me/proj` | `ADMISSION_HOOK_PODMAN_PREFIX` |

  Both `\` and `/` separators are accepted, as are `\\?\C:\...` paths. POSIX
  paths and volume names are left untouched.

* **Startup order.** The hook
  binary is the entrypoint:
  1. It binds and serves the hook on `127.0.0.1:12777`.
  2. It registers the hook in the gateway config. It takes the file given by
     `--config`, `OPENSHELL_GATEWAY_CONFIG`, or the gateway's XDG default,
     copies it, appends the `[[openshell.gateway.interceptors]]` block below,
     and writes the result to `/tmp/openshell-admission-hook/gateway.toml`.
     Your file is never modified. If no config exists, it generates a minimal
     one.
  3. It starts `/usr/local/bin/openshell-gateway` with the container
     arguments, pointing it at the generated config.
  4. It forwards SIGTERM, SIGINT, SIGQUIT and SIGHUP, reaps zombies, and exits
     with the gateway's exit code. If the hook dies, it stops the gateway, so
     the container never runs without the hook.

  Injected registration:

  ```toml
  [[openshell.gateway.interceptors]]
  name = "windows-mount-admission"
  grpc_endpoint = "http://127.0.0.1:12777"
  allow_insecure_transport = true   # loopback inside the container only
  order = -1000                     # runs before other interceptors
  failure_policy = "fail_closed"
  binding_policy = "exact"
  timeout = "2s"

  [[openshell.gateway.interceptors.bindings]]
  rpc = "openshell.v1.OpenShell/CreateSandbox"
  phases = ["modify_operation"]
  ```

  At startup the gateway logs one warning about `allow_insecure_transport`.
  This is expected: the hook only listens on loopback inside the gateway
  container.

## Build

```sh
cargo build 
```

### OpenShell protos

The protobuf definitions aren't vendored. The version in
[`openshell.version`](openshell.version) selects which ones to use. It is a
bare version such as `0.1.9`, with no leading `v`, the same form as the
gateway image tag. At build time, `crates/path-admission/build.rs`:

1. downloads `gateway_interceptor.proto` and `extension.proto` from
   `https://raw.githubusercontent.com/NVIDIA/OpenShell/v<version>/proto/`, plus every
   non-`google/` file they import, recursively;
2. caches them per tag under `target/<profile>/openshell-proto/<tag>/`;
3. compiles them with [`protox`](https://crates.io/crates/protox) (a pure-Rust
   protobuf compiler with the well-known types built in, so no system `protoc`
   is needed), then generates the code with `tonic-prost-build`.

To move to another OpenShell release, edit `openshell.version` and rebuild.
The gateway interceptor protocol must have the same major version. Overrides:

| Env var | Purpose |
|---|---|
| `OPENSHELL_PROTO_DIR` | use a local `proto/` directory (offline / OpenShell checkout); no download |
| `OPENSHELL_PROTO_BASE_URL` | mirror instead of `https://raw.githubusercontent.com/NVIDIA/OpenShell` |

The build runs the unit tests (`cargo test`). The crate is Unix-only because it
uses signals and `waitpid`. On Windows, build it through the container.

### CI

| Workflow | Trigger | Does |
|---|---|---|
| `pr-check.yaml` | pull requests | `cargo fmt --check`, `cargo clippy -D warnings` |
| `next-build.yaml` | push to `main`, manual | build and tests |


Dependabot updates the GitHub Actions and the cargo dependencies (tonic and
prost grouped).


### Settings

| Env var | Default | Purpose |
|---|---|---|
| `ADMISSION_HOOK_LISTEN` | `127.0.0.1:12777` | hook listen address |
| `ADMISSION_HOOK_ENDPOINT` | `http://<listen>` | endpoint written into the gateway config |
| `ADMISSION_HOOK_DOCKER_PREFIX` | `/run/desktop/mnt/host` | Docker drive prefix |
| `ADMISSION_HOOK_PODMAN_PREFIX` | `/mnt` | Podman drive prefix |
| `ADMISSION_HOOK_TIMEOUT` | `2s` | interceptor RPC timeout (10ms to 30s) |
| `ADMISSION_HOOK_SKIP_CONFIG_INJECTION` | unset | `true`: don't touch the config; register the hook yourself |
| `ADMISSION_HOOK_RUNTIME_DIR` | `/tmp/openshell-admission-hook` | where the generated config is written |
| `ADMISSION_HOOK_LOG` | `info` | log filter (`debug` logs every evaluation) |

`openshell-admission-hook serve` runs the hook alone, without a gateway.

## Layout

Cargo workspace; the admission hook is the `crates/path-admission` crate
(binary `openshell-admission-hook`).

| Path | |
|---|---|
| `crates/path-admission/src/paths.rs` | Windows path to machine path conversion |
| `crates/path-admission/src/hook.rs` | gRPC interceptor (Describe negotiation, Evaluate, patches) |
| `crates/path-admission/src/launcher.rs` | entrypoint: config injection and gateway supervision |
| `crates/path-admission/build.rs` | downloads and compiles the OpenShell protos for the pinned tag |
| `openshell.version` | OpenShell version (e.g. `0.1.9`) the protos are taken from (git tag `v<version>`) |
