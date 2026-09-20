# First-time setup

This guide gets one GitHub repository running on Loadward using the simplest supported execution path:

```text
GitHub Actions workflow
        |
        v
GitHub App installation
        |
        v
Loadward daemon
        |
        v
isolated-local profile
        |
        v
Docker container
```

The GitHub App credentials stay on the machine running the daemon. They are **not** committed to the repository and are **not** added as repository Actions secrets.

## 1. Prerequisites

On the Linux machine that will run the daemon, install and verify Docker:

```bash
docker version
docker run --rm hello-world
```

Install the Loadward binary from this repository's Releases page as described in [README.md](README.md), then verify it is on your path:

```bash
loadward --help
```

## 2. Create a GitHub App

Create a separate GitHub App for the Loadward installation that you control.

In GitHub:

1. Open **Settings → Developer settings → GitHub Apps**.
2. Select **New GitHub App**.
3. Give it a unique name, for example `loadward-<your-login>`.
4. Set **Homepage URL** to `https://github.com/incirci/loadward-dist` or another URL you control.
5. Leave user authorization/callback settings unused.
6. Under **Webhooks**, turn **Active** off. Loadward does not require GitHub App webhooks.
7. Under **Repository permissions**, set exactly:
   - **Actions: Read and write**
   - **Administration: Read and write**
   - **Issues: Read and write**
8. Leave other repository, organization, and account permissions at their defaults.
9. Under **Where can this GitHub App be installed?**, choose **Only on this account** unless you specifically need to install the same app on another account or organization.
10. Create the app.

Why these permissions:

- **Administration** is required to manage repository-scoped self-hosted runner scale sets.
- **Actions** is required for GitHub Actions integration and Loadward's GitHub execution ingress.
- **Issues** is used by Loadward's owner-request issue ingress.

GitHub recommends granting a GitHub App only the permissions it needs.

## 3. Record the Client ID and generate a private key

On the new GitHub App's settings page, copy its **Client ID**. Loadward expects the Client ID, not the numeric App ID.

Then scroll to **Private keys** and select **Generate a private key**. GitHub downloads a `.pem` file.

Move that key to the daemon host configuration directory:

```bash
mkdir -p ~/.config/loadward
chmod 700 ~/.config/loadward
mv ~/Downloads/*.pem ~/.config/loadward/github-app.pem
chmod 600 ~/.config/loadward/github-app.pem
```

If there is more than one `.pem` file in `~/Downloads`, move the correct GitHub App key explicitly rather than using the wildcard.

Never commit this key.

## 4. Install the GitHub App on the repository

From the GitHub App settings page:

1. Select **Install App**.
2. Choose the user or organization that owns the repository.
3. Prefer **Only select repositories**.
4. Select the repository Loadward should serve.
5. Complete the installation.

Then open **Configure** for that installed app. The browser URL contains `/installations/<number>`. That number is the **installation ID** Loadward needs.

For example, if the URL ends in:

```text
/installations/12345678
```

then:

```text
app_installation_id = 12345678
```

All repositories configured as targets for this daemon must be accessible through the configured GitHub App installation.

If you later change the GitHub App's requested permissions, GitHub may require the installation owner to approve the updated permissions before new installation tokens receive them.

## 5. Create the Loadward configuration

Create:

```text
~/.config/loadward/config.toml
```

Start with the minimal Docker configuration below, replacing the four uppercase placeholders:

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

[[targets]]
name = "YOUR_REPOSITORY_NAME"
url = "https://github.com/YOUR_OWNER/YOUR_REPOSITORY_NAME"
profiles = ["isolated-local"]
```

You can also copy [`examples/config.toml`](examples/config.toml).

Use the repository basename for `name`. For example:

```toml
[[targets]]
name = "my-project"
url = "https://github.com/alice/my-project"
profiles = ["isolated-local"]
```

The target is an explicit allowlist. A repository can request only the profiles listed for that target.

`max_runners` is the GitHub adapter ceiling. `capacity` is the shared Loadward capacity for that execution profile. They are separate limits.

## 6. Validate the configuration

Before starting the daemon:

```bash
loadward -check
```

Expected:

```text
configuration valid
```

If validation fails, fix the configuration before continuing. Loadward intentionally rejects unknown or obsolete configuration fields rather than silently accepting aliases or fallback behavior.

## 7. Run the daemon as a user service

Create the systemd user service:

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

## 8. Verify the daemon can see the repository

Run:

```bash
loadward doctor --target YOUR_REPOSITORY_NAME
```

A healthy idle route should report the listener and provider as healthy/ready. Demand-scaled GitHub runners may correctly show zero active runners while no workflow is waiting.

If the target is absent, check:

- the repository URL in `config.toml`;
- that the GitHub App is installed on that repository;
- that the configured installation ID belongs to that installation;
- that the daemon was restarted after changing the configuration.

## 9. Add a workflow to the repository

In the repository that should use Loadward, add:

```text
.github/workflows/loadward-smoke.yml
```

with:

```yaml
name: Loadward smoke

on:
  workflow_dispatch:

jobs:
  smoke:
    runs-on: isolated-local
    timeout-minutes: 5
    steps:
      - name: Verify runner
        run: |
          echo "Loadward runner is alive"
          uname -a
          id
```

A copy is available at [`examples/loadward-smoke.yml`](examples/loadward-smoke.yml).

The important line is:

```yaml
runs-on: isolated-local
```

The value must exactly match a profile exposed by that repository's `[[targets]]` entry. Do not add `self-hosted` or another label unless a future Loadward contract explicitly requires it.

No Loadward GitHub App private key or installation token belongs in this workflow.

## 10. Run the smoke test

In GitHub:

1. Open the repository's **Actions** tab.
2. Open **Loadward smoke**.
3. Select **Run workflow**.

While the job is running, you can inspect the daemon with:

```bash
loadward doctor --target YOUR_REPOSITORY_NAME
journalctl --user -u loadward.service -f
```

The expected lifecycle is:

```text
workflow queued
    -> Loadward detects demand
    -> execution resource is created
    -> ephemeral GitHub runner accepts the job
    -> job runs
    -> runner/resource is cleaned up
    -> route returns to idle
```

After the job completes, `doctor` can legitimately return to zero provider/GitHub runners because the execution path is demand-scaled.

## Adding another repository

For another repository under the same GitHub App installation:

1. Open the installed GitHub App's **Configure** page and add the repository to its repository access.
2. Add another target to `~/.config/loadward/config.toml`:

```toml
[[targets]]
name = "another-repo"
url = "https://github.com/YOUR_OWNER/another-repo"
profiles = ["isolated-local"]
```

3. Validate and restart:

```bash
loadward -check
systemctl --user restart loadward.service
loadward doctor --target another-repo
```

The same daemon can serve multiple repositories through the same GitHub App installation.

## Common failures

| Symptom | Check first |
| --- | --- |
| `401` / authentication failure | Client ID, installation ID, and matching private key |
| `403` from GitHub | GitHub App permissions and whether updated permissions were approved |
| target missing from `doctor` | `[[targets]]` name/URL and daemon restart |
| listener unhealthy | daemon logs and GitHub App installation access |
| workflow remains queued | exact `runs-on` profile, daemon health, target profile allowlist |
| Docker execution fails | `docker version`, daemon access, and pulling `ghcr.io/actions/actions-runner:latest` |

Useful commands:

```bash
loadward -check
loadward status --target YOUR_REPOSITORY_NAME
loadward doctor --target YOUR_REPOSITORY_NAME
journalctl --user -u loadward.service --no-pager -n 200
```

## Security boundary

The minimal setup above uses `isolated-local`, which runs the GitHub job in a disposable Docker execution environment rather than directly as the daemon host user.

Only enable broader execution profiles for repositories you trust and only when you intentionally need that stronger access boundary.
