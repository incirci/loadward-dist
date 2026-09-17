# RunMeSome Distribution

Public binary distribution repository for RunMeSome.

This repository intentionally contains no RunMeSome source code. Packaged release binaries are published here through **Releases**.

## Download

No GitHub account or source-repository access is required to download a public release.

### Linux x86-64

```bash
mkdir -p /tmp/runmesome-install
cd /tmp/runmesome-install

curl -fL \
  -o runmesome-linux-amd64 \
  https://github.com/incirci/runmesome-dist/releases/latest/download/runmesome-linux-amd64

curl -fL \
  -o SHA256SUMS \
  https://github.com/incirci/runmesome-dist/releases/latest/download/SHA256SUMS

grep ' runmesome-linux-amd64$' SHA256SUMS | sha256sum -c -
chmod +x runmesome-linux-amd64
install -Dm755 runmesome-linux-amd64 ~/.local/bin/runmesome
```

For ARM64, replace `amd64` with `arm64`.

If GitHub CLI is already installed:

```bash
gh release download \
  --repo incirci/runmesome-dist \
  --pattern runmesome-linux-amd64 \
  --pattern SHA256SUMS
```

## First-time GitHub setup

Downloading the binary is only the first step. To let the RunMeSome daemon serve a GitHub repository, you must also:

1. create a GitHub App;
2. install that app on the repository;
3. store the app private key on the daemon host;
4. configure the GitHub installation and repository target in RunMeSome;
5. run the daemon;
6. use a RunMeSome execution profile in the repository workflow.

Follow **[SETUP.md](SETUP.md)** for the complete start-to-finish procedure.

A minimal ready-to-edit daemon configuration is in [`examples/config.toml`](examples/config.toml), and a repository smoke workflow is in [`examples/runmesome-smoke.yml`](examples/runmesome-smoke.yml).

## Releases

Releases are created only by an explicitly triggered release workflow in the private RunMeSome source repository. Ordinary commits and merges do not publish releases.

Each release contains:

- `runmesome-linux-amd64`
- `runmesome-linux-arm64`
- `SHA256SUMS`
- `BUILDINFO.txt`

`BUILDINFO.txt` records the release version and exact private-source commit used for the build.

## Repository purpose

This repository is intentionally public so release binaries and deployment instructions can be used without granting access to the private source repository.

Development, source history, CI internals, and unreleased code remain in the private source repository.
