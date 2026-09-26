# Loadward setup

This guide gets the public Loadward binary running with the current pool/backend architecture and, optionally, GitHub Actions.

Loadward itself is not GitHub-only. Direct CLI execution needs only execution pools; the GitHub sections are optional.

## 1. Install Loadward

Follow the download instructions in [`README.md`](README.md), then verify:

```bash
loadward version
loadward --help
```

Use `loadward help <command>` for command-specific help.

## 2. Choose the execution pools you need

Execution pools are the configured capacity/backend objects shared by direct workloads and adapters such as GitHub Actions.

Each pool owns:

- one stable operational identity;
- one backend implementation and its settings;
- one shared admission capacity.

Pool names do not define privileges. Loadward derives capabilities from backend configuration and matches CLI/MCP requests and GitHub routes against those capabilities.

There is no separate configured provider object and no configured profile layer.

Typical backends are:

| Backend | Purpose | Main prerequisite |
| --- | --- | --- |
| `docker` | isolated CPU/GPU containers and controlled workspace access | Docker |
| `host` | direct execution as the Loadward user | trusted host execution |
| `libvirt` | disposable desktop VM | KVM/QEMU + libvirt + prepared QCOW2 |
| `runpod-serverless` | remote CPU/GPU execution | RunPod API key + compatible immutable worker image |

## 3. Create and install a GitHub App when GitHub Actions is needed

Skip this section for direct-run-only installations.

Create a GitHub App owned by the account or organization that owns the repositories Loadward should serve.

Use these repository permissions:

```text
Actions:        Read and write
Administration: Read and write
Issues:         Read and write
```

Store the generated private key outside any repository, for example:

```text
~/.config/loadward/github-app.pem
```

Install the App on the repositories Loadward should serve. The GitHub App installation establishes the trusted repository set; Loadward discovers those repositories automatically. There is no repository list in `config.toml`.

Record the App client ID, installation ID, and private-key path. If installation scope or App permissions change, approve the updated installation access and restart Loadward.

## 4. Create the configuration

Create `~/.config/loadward/config.toml`.

A minimal local-container GitHub configuration is:

```toml
[github]
app_client_id = "YOUR_GITHUB_APP_CLIENT_ID"
app_installation_id = 12345678
private_key_file = "~/.config/loadward/github-app.pem"
# Optional: enable daemon-native issue ingress on one exact installed repository.
# exec_issue_repository = "OWNER/REPOSITORY"
max_runners = 1

[pools.isolated-local]
backend = "docker"
capacity = 1
image = "ghcr.io/actions/actions-runner:latest"

[github.routes.local-isolated]
isolation = "container"
workspace = "none"
location = "local"
```

You can copy [`examples/config.toml`](examples/config.toml).

`github.exec_issue_repository` is the explicit opt-in for ChatGPT/ad-hoc issue ingress. It must be one exact installed `owner/repository` identity. Omit it to keep issue ingress disabled; Loadward never infers a control repository from its basename.

The ownership layers are intentionally different:

```text
GitHub workflow -> route -> placement/admission -> execution pool -> backend
```

- `local-isolated` is a GitHub-visible route.
- `isolated-local` is an internal pool identity.
- `docker` is the selected pool's backend implementation.

Workflows select routes, never pool names.

`github.max_runners` is an adapter-side ceiling for each repository/route scale set. Pool `capacity` is the physical admission limit shared by GitHub, direct CLI, MCP/local agents, and other consumers.

A direct-run-only configuration may omit the entire `[github]` section and every `[github.routes.*]` entry.

## 5. Configure authorization for privileged contracts

Authorization is separate from pool capability.

A typical policy is:

```toml
[authorization.github]
host = "allow"
workspace-read = "allow"
workspace-write = "allow"
cloud = "allow"
gpu = "allow"
desktop-session = "allow"

[authorization.local-control]
host = "require-grant"
workspace-read = "allow"
workspace-write = "allow"
cloud = "allow"
gpu = "allow"
desktop-session = "allow"
```

Supported modes are `allow`, `require-grant`, and `deny`.

Authorization is evaluated against the concrete effective execution contract. A missing or denied privileged gate fails closed. `require-grant` keeps the capability available in principle but blocks new admission until the exact trusted principal has an active temporary grant.

Examples:

```bash
loadward authorization list

# local control-socket principal
loadward authorization grant \
  --kind local-control \
  --id local-user \
  --gate host \
  --ttl 30m

# one exact GitHub repository principal
loadward authorization grant \
  --kind github \
  --id incirci/my-project \
  --gate gpu \
  --ttl 2h
```

Temporary grants live in daemon memory and expire automatically.

## 6. Configure execution pools

### Local Docker

```toml
[pools.isolated-local]
backend = "docker"
capacity = 4
image = "ghcr.io/actions/actions-runner:latest"
```

The normal host home directory and Docker socket are not mounted.

### Local Docker with GPU

Prerequisites: Docker plus a working NVIDIA driver/runtime.

```toml
[pools.isolated-local-gpu]
backend = "docker"
capacity = 1
image = "YOUR_GPU_RUNNER_IMAGE"
gpus = "all"
```

GPU is a fixed pool specialization. A request that uses that capacity is authorized against the resulting GPU-capable effective contract.

### Docker with controlled host workspace

One pool can realize several workspace contracts while sharing one capacity domain:

```toml
[pools.local-container]
backend = "docker"
capacity = 4
image = "ghcr.io/actions/actions-runner:latest"
workspace_root = "~/Code"
workspace_modes = ["none", "read-only", "read-write"]
```

The selected effective contract determines whether `/workspace` is absent, read-only, or read-write. Do not create separate RO/RW pools merely to represent access modes of the same physical capacity domain.

### Direct host execution

```toml
[pools.host-local]
backend = "host"
capacity = 1
```

The host backend runs workloads directly as the Linux user running Loadward. It is intentionally privileged.

If a GitHub route can select host execution, `[github].runner_dir` must point to an unregistered official GitHub Actions runner installation used as the immutable runner template:

```toml
[github]
runner_dir = "~/.local/share/loadward/actions-runner"
```

Host isolation and host workspace are never selected implicitly.

### Libvirt desktop VM

Prerequisites include KVM/QEMU, libvirt user-session access, `systemd-run`, and either `virtqemud` or `libvirtd`.

```bash
command -v systemd-run
command -v virtqemud || command -v libvirtd
```

Provide a compatible desktop QCOW2 base image, then configure:

```toml
[pools.desktop-local]
backend = "libvirt"
capacity = 4
base_image = "~/.local/share/loadward/images/ubuntu-24.04-desktop.qcow2"
libvirt_uri = "qemu:///session"
state_dir = "~/.local/state/loadward/desktop-vm"
memory_mb = 8192
vcpus = 4
video_heads = 2
```

The public binary distribution does not include a ready-made desktop base image.

### RunPod Serverless

Store the RunPod API key outside the repository:

```bash
mkdir -p ~/.config/loadward
printf '%s\n' '<RUNPOD_API_KEY>' > ~/.config/loadward/runpod-api-key
chmod 600 ~/.config/loadward/runpod-api-key
```

Loadward requires an explicit immutable worker artifact. Configure the repository digest of a compatible worker image:

```toml
[runpod]
worker_image = "ghcr.io/OWNER/IMAGE@sha256:REPLACE_WITH_64_HEX_DIGEST"
# container_registry_auth_id = "RUNPOD_REGISTRY_CREDENTIAL_ID"
```

Loadward does not infer a worker image from an existing endpoint or clone a hidden bootstrap template. It reconciles each managed template/endpoint to the configured worker artifact.

CPU pool:

```toml
[pools.isolated-cloud]
backend = "runpod-serverless"
capacity = 4
compute = "cpu"
api_key_file = "~/.config/loadward/runpod-api-key"
```

GPU pool:

```toml
[pools.isolated-cloud-gpu]
backend = "runpod-serverless"
capacity = 1
compute = "gpu"
gpu_type = "NVIDIA A40"
api_key_file = "~/.config/loadward/runpod-api-key"
```

Optional per-pool overrides include:

```toml
execution_timeout_seconds = 3600
job_ttl_seconds = 7200
```

There is no separate `max_workers`; RunPod worker capacity derives from `pool.capacity`.

The public distribution does not currently ship the compatible RunPod worker image itself.

## 7. Configure GitHub routes

Routes express execution requirements and are global across repositories in the GitHub App installation.

Portable routes can rank local before cloud:

```toml
[github.routes.isolated]
isolation = "container"
workspace = "none"
prefer_locations = ["local", "cloud"]

[github.routes.gpu]
isolation = "container"
workspace = "none"
features = ["gpu"]
prefer_locations = ["local", "cloud"]
```

Hard-location routes are also supported:

```toml
[github.routes.local-isolated]
isolation = "container"
workspace = "none"
location = "local"

[github.routes.cloud-isolated]
isolation = "container"
workspace = "none"
location = "cloud"
```

Other capability routes:

```toml
[github.routes.workspace-ro]
isolation = "container"
workspace = "read-only"

[github.routes.workspace-rw]
isolation = "container"
workspace = "read-write"

[github.routes.host]
isolation = "host"
workspace = "host"

[github.routes.desktop]
isolation = "vm"
workspace = "none"
features = ["desktop-session"]
```

`prefer_locations` changes ranking only. `location` is a hard requirement.

Routes remain global, but privileged admission is authorization-aware. Permanent deny/missing-gate policy removes impossible capacity; `require-grant` contributes no free capacity until the exact repository principal has a matching active grant.

Configuration is forward-only. Legacy provider/profile configuration and old repository-to-profile mappings are rejected rather than translated through compatibility aliases.

## 8. Validate the configuration

```bash
loadward --check
```

Expected:

```text
configuration valid
```

Use `--config /path/to/config.toml` to validate another file.

## 9. Run the daemon as a user service

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/loadward.service <<'EOF'
[Unit]
Description=Loadward execution environment manager

[Service]
Type=simple
ExecStart=%h/.local/bin/loadward
KillMode=control-group
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now loadward.service
systemctl --user status loadward.service --no-pager
```

Follow logs with:

```bash
journalctl --user -u loadward.service -f
```

## 10. Verify the running system

Bare status is fleet-wide:

```bash
loadward status
loadward status --summary
```

For one discovered repository:

```bash
loadward status --target my-project
loadward doctor --target my-project
```

`doctor` deliberately requires an explicit scope. Use `--target` and/or `--route`, or explicit `--all`.

A healthy idle demand-scaled route can legitimately have zero backend/GitHub runners.

If a repository is absent, check the GitHub App installation scope, installation ID, permission approval, and whether Loadward was restarted after scope changed.

## 11. Add a smoke workflow

Add `.github/workflows/loadward-smoke.yml`:

```yaml
name: Loadward smoke

on:
  workflow_dispatch:

jobs:
  smoke:
    runs-on: local-isolated
    timeout-minutes: 5
    steps:
      - name: Verify runner
        run: |
          echo "Loadward runner is alive"
          uname -a
          id
```

A copy is available at [`examples/loadward-smoke.yml`](examples/loadward-smoke.yml).

The `runs-on` value is the configured **route**. Do not use pool names, backend names, `self-hosted`, OS labels, or retired compatibility labels.

The expected lifecycle is:

```text
workflow queued
    -> Loadward observes route demand
    -> route requirements are matched against authorized compatible pools
    -> one pool slot is atomically admitted
    -> backend resource + ephemeral runner are created
    -> job runs
    -> runner/resource are cleaned up
    -> pool capacity returns to idle
```

Inspect with:

```bash
loadward doctor --target my-project --route local-isolated
journalctl --user -u loadward.service -f
```

## Adding another repository

Add the repository to the same GitHub App installation and restart Loadward. No TOML repository entry is required.

Every configured GitHub route is visible to every installed repository, but privileged admission is still evaluated against the exact repository principal and current authorization policy/grants.

## Common failures

| Symptom | Check first |
| --- | --- |
| `401` / authentication failure | Client ID, installation ID, matching private key |
| `403` from GitHub | App permissions and approval of updated permissions |
| repository missing from `status` / `doctor` | App installation scope and daemon restart |
| listener unhealthy | daemon logs and GitHub App installation access |
| workflow remains queued | exact `runs-on` route, authorization, compatible pool capacity/backend health |
| Docker execution fails | `docker version`, daemon access, image pull |
| RunPod pool unavailable | API key, immutable `runpod.worker_image`, template/endpoint reconciliation |

Useful commands:

```bash
loadward --check
loadward status --summary
loadward doctor --target my-project
loadward authorization list
loadward recovery status
journalctl --user -u loadward.service --no-pager -n 200
```

## Security boundary

Use the narrowest environment that satisfies the workload.

GitHub App installation membership defines which repositories are trusted principals. Authorization then governs privileged effective contracts such as host, workspace, cloud, GPU, and desktop-session. A route name never bypasses those checks.
