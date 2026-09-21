# Loadward setup

This guide gets GitHub Actions running on Loadward using the current route-based contract.

Loadward itself is not GitHub-only. Direct CLI execution needs only providers and profiles; the GitHub section is optional.

## 1. Install Loadward

Follow the download instructions in [`README.md`](README.md), then verify:

```bash
loadward version
loadward --help
```

## 2. Install prerequisites

For the minimal local-container setup:

```bash
docker version
```

The Linux user running Loadward must be able to use Docker.

## 3. Create and install a GitHub App

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

Install the App on the repositories Loadward should serve. The **GitHub App installation is the repository authorization boundary**: Loadward discovers those repositories automatically. There is no repository list or per-repository profile allowlist in `config.toml`.

Record the App client ID, installation ID, and private-key path. If installation scope or App permissions change, approve the updated installation access and restart Loadward.

## 4. Create the configuration

Create `~/.config/loadward/config.toml`.

A minimal local-container configuration is:

```toml
[github]
app_client_id = "YOUR_GITHUB_APP_CLIENT_ID"
app_installation_id = 12345678
private_key_file = "~/.config/loadward/github-app.pem"
max_runners = 1

[providers.local-docker-isolated]
type = "docker"
image = "ghcr.io/actions/actions-runner:latest"

[profiles.isolated-local]
provider = "local-docker-isolated"
capacity = 1
priority = 10

[github.routes.local-isolated]
isolation = "container"
workspace = "none"
location = "local"
```

You can copy [`examples/config.toml`](examples/config.toml).

The identity layers are intentionally different:

```text
GitHub workflow -> route -> placement -> internal profile -> provider
```

- `local-isolated` is a stable GitHub-visible route.
- `isolated-local` is an internal profile identity.
- `local-docker-isolated` is a provider identity.

Workflows never select the internal profile or provider.

`github.max_runners` is an adapter-side ceiling. Profile `capacity` is the shared physical admission limit used by GitHub, direct CLI, MCP/local agents, and other consumers.

## 5. Validate the configuration

```bash
loadward --check
```

Expected:

```text
configuration valid
```

Current configuration is forward-only. Legacy repository-target mappings are rejected rather than silently accepted.

## 6. Run the daemon as a user service

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

## 7. Verify repository discovery

For an installed repository named `my-project`:

```bash
loadward status --target my-project
loadward doctor --target my-project
```

A healthy idle route can legitimately have zero provider/GitHub runners because capacity is demand-scaled.

If the repository is absent, check the GitHub App installation scope, installation ID, permission approval, and whether Loadward was restarted after scope changed.

## 8. Add a smoke workflow

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

The `runs-on` value is the configured **route**. Do not use `isolated-local`, provider names, `self-hosted`, OS labels, or compatibility labels.

## 9. Run the smoke test

Run **Loadward smoke** from the repository's Actions tab.

The expected lifecycle is:

```text
workflow queued
    -> Loadward observes route demand
    -> planner selects a compatible profile
    -> profile capacity is admitted
    -> execution resource + ephemeral runner are created
    -> job runs
    -> runner/resource are cleaned up
    -> route returns to idle
```

Inspect with:

```bash
loadward doctor --target my-project --route local-isolated
journalctl --user -u loadward.service -f
```

## Adding another repository

Add the repository to the same GitHub App installation and restart Loadward. No TOML repository entry is required.

Every configured GitHub route is available to every repository in that App installation. Use a separate App installation if you need a different repository trust boundary.

## Common failures

| Symptom | Check first |
| --- | --- |
| `401` / authentication failure | Client ID, installation ID, matching private key |
| `403` from GitHub | App permissions and approval of updated permissions |
| repository missing from `status` / `doctor` | App installation scope and daemon restart |
| listener unhealthy | daemon logs and GitHub App installation access |
| workflow remains queued | exact `runs-on` route and compatible profile/provider health |
| Docker execution fails | `docker version`, daemon access, image pull |

Useful commands:

```bash
loadward --check
loadward status
loadward doctor --target my-project
journalctl --user -u loadward.service --no-pager -n 200
```

## Security boundary

The minimal `local-isolated` route maps to a disposable local Docker environment. Stronger routes such as host, writable workspace, desktop, or GPU execution should be configured only when intentionally required.

GitHub App installation membership is the repository trust boundary. There is intentionally no per-repository route authorization layer inside Loadward.
