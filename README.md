# Loadward Distribution

Public binary distribution repository for Loadward.

This repository intentionally contains no Loadward source code. Packaged release binaries are published here through **Releases**.

## Download

No GitHub account or source-repository access is required to download a public release.

> **Rename note:** releases published before the RunMeSome → Loadward rename are historical artifacts and keep their original release titles and `runmesome-linux-*` asset names. GitHub does not retroactively rename release assets. The `loadward-linux-*` commands below apply to Loadward-branded releases produced by the current release workflow. If `releases/latest` still points at a historical RunMeSome release, publish or select a post-rename Loadward release before using these asset URLs.

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

## Direct execution

GitHub is optional for Loadward's core execution path. A daemon configured only with providers and profiles can explain and run workloads directly:

```bash
loadward explain
loadward run -- /bin/sh -c 'echo ok'
```

Direct execution uses requirement-based placement by default. Specialized boundaries are requested explicitly, for example:

```bash
loadward explain --desktop-session
loadward run --gpu --prefer-location cloud -- ./scripts/gpu-proof
```

An explicit `--profile` is a strict override, not a privilege or capability bypass.

## First-time GitHub setup

Downloading the binary is only the first step if you want GitHub Actions integration. To let the Loadward daemon serve a GitHub repository, you must also:

1. create a GitHub App;
2. install that app on the repository;
3. store the app private key on the daemon host;
4. configure the GitHub installation and repository target in Loadward;
5. run the daemon;
6. use a Loadward execution profile in the repository workflow.

Follow **[SETUP.md](SETUP.md)** for the complete start-to-finish procedure.

A minimal ready-to-edit daemon configuration is in [`examples/config.toml`](examples/config.toml), and a repository smoke workflow is in [`examples/loadward-smoke.yml`](examples/loadward-smoke.yml).

## ChatGPT integration

For binary-only deployments, Loadward can also accept constrained ChatGPT execution requests through its GitHub issue ingress without exposing the daemon to the internet. Source-built deployments additionally expose the `loadward-mcp` adapter; the private source repository documents that MCP path separately.

ChatGPT uses its own connected GitHub account to create a constrained execution issue in a control repository. The Loadward daemon consumes that issue through its separate GitHub App connection and dispatches the canonical execution workflow.

Follow **[CHATGPT.md](CHATGPT.md)** for the complete setup, including:

- connecting GitHub to ChatGPT;
- configuring the `loadward` control target;
- installing [`examples/exec.yml`](examples/exec.yml) in the control repository;
- the exact issue protocol ChatGPT must use;
- profile and security boundaries;
- an end-to-end ChatGPT smoke test.

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
