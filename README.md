# Loadward Distribution

Public binary distribution repository for Loadward.

This repository intentionally contains no Loadward source code. Packaged release binaries are published here through **Releases**.

Loadward is an execution-resource manager. Callers describe the environment they need; Loadward matches that request against configured execution **pools**, admits shared capacity, starts the pool's **backend**, observes the execution, and cleans it up. GitHub Actions is one adapter to that core rather than the core abstraction itself.

## Download

No GitHub account or source-repository access is required to download a public release.

> **Rename note:** releases published before the RunMeSome → Loadward rename are historical artifacts and keep their original release titles and `runmesome-linux-*` asset names. GitHub does not retroactively rename release assets. The `loadward-linux-*` commands below apply to Loadward-branded releases produced by the current release workflow.

### Linux x86-64

```bash
mkdir -p /tmp/loadward-install
cd /tmp/loadward-install

curl -fL \
  -o loadward-linux-amd64 \
  https://github.com/incirci/loadward-dist/releases/latest/download/loadward-linux-amd64

curl -fL \
  -o SHA256SUMS \
  https://github.com/incirci/loadward-dist/releases/latest/download/SHA256SUMS

grep ' loadward-linux-amd64$' SHA256SUMS | sha256sum -c -
chmod +x loadward-linux-amd64
install -Dm755 loadward-linux-amd64 ~/.local/bin/loadward
```

For ARM64, replace `amd64` with `arm64`.

If GitHub CLI is already installed:

```bash
gh release download \
  --repo incirci/loadward-dist \
  --pattern loadward-linux-amd64 \
  --pattern SHA256SUMS
```

Verify the installed binary and discover the current CLI:

```bash
loadward version
loadward --help
loadward help run
```

## Direct execution

GitHub is optional. A direct-run-only daemon needs only execution pools:

```toml
[pools.default]
backend = "docker"
capacity = 1
image = "ubuntu:24.04"
```

Placement is requirement-based:

```bash
loadward explain
loadward run -- /bin/sh -c 'echo ok'

loadward explain --desktop-session
loadward run --gpu --prefer-location cloud -- ./scripts/gpu-proof
```

Pool names are operational identities, not placement privileges. There is no configured provider/profile layer and no `--profile` placement override.

## Operational CLI

The built-in help is the source of truth for the installed binary:

```bash
loadward --help
loadward help status
loadward help doctor
loadward help authorization
loadward help recovery
```

Useful read-only commands:

```bash
# fleet-wide status
loadward status
loadward status --summary

# doctor deliberately requires explicit scope
loadward doctor --target my-project
loadward doctor --all
```

`status` is fleet-wide when no selector is supplied. `doctor` is intentionally different: it requires `--target` and/or `--route`, or explicit `--all`.

## First-time GitHub setup

Downloading the binary is only the first step if you want GitHub Actions integration. To let the Loadward daemon serve GitHub repositories:

1. create a GitHub App;
2. install that app on the repositories Loadward should serve;
3. store the app private key on the daemon host;
4. configure execution pools, authorization policy when needed, and GitHub routes;
5. run the daemon;
6. use a configured Loadward route as the workflow's single `runs-on` label.

Follow **[SETUP.md](SETUP.md)** for the complete start-to-finish procedure.

A minimal current configuration is in [`examples/config.toml`](examples/config.toml), and a repository smoke workflow is in [`examples/loadward-smoke.yml`](examples/loadward-smoke.yml).

## ChatGPT integration

For binary-only deployments, Loadward can accept constrained ChatGPT execution requests through GitHub issue ingress without exposing the daemon to the internet.

ChatGPT uses its connected GitHub account to create a constrained execution issue in a control repository. The Loadward daemon consumes that issue through its separate GitHub App connection and dispatches the canonical execution workflow. The requested route is resolved through the same authorization, placement, admission, pool, and backend machinery as any other GitHub workload.

Follow **[CHATGPT.md](CHATGPT.md)** for the complete setup.

No ChatGPT token, OpenAI API key, inbound daemon port, or Loadward private key is required for this bridge.

## Releases

Releases are created only by an explicitly triggered release workflow in the private Loadward source repository. Ordinary commits and merges do not publish releases.

Current Loadward-branded releases contain:

- `loadward-linux-amd64`
- `loadward-linux-arm64`
- `SHA256SUMS`
- `BUILDINFO.txt`

`BUILDINFO.txt` records the release version and exact private-source commit used for the build.

## Repository purpose

This repository is intentionally public so release binaries and deployment instructions can be used without granting access to the private source repository.

Development, source history, CI internals, and unreleased code remain in the private source repository.
