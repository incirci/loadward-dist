# RunMeSome Distribution

Public binary distribution repository for RunMeSome.

This repository intentionally contains no RunMeSome source code. The source repository remains private; packaged release binaries are published here through **Releases**.

## Download

No GitHub account or repository access is required to download a public release.

### Browser

Open **Releases**, choose the required version, and download the binary for your Linux architecture.

### Command line

Download the latest x86-64 build and checksums without authentication:

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

If GitHub CLI is already installed, the equivalent download is:

```bash
gh release download \
  --repo incirci/runmesome-dist \
  --pattern runmesome-linux-amd64 \
  --pattern SHA256SUMS
```

## Releases

Releases are created only by an explicitly triggered release workflow in the private RunMeSome source repository. Ordinary commits and merges do not publish releases.

Each release contains:

- `runmesome-linux-amd64`
- `runmesome-linux-arm64`
- `SHA256SUMS`
- `BUILDINFO.txt`

`BUILDINFO.txt` records the release version and exact private-source commit used for the build.

## Repository purpose

This repository is intentionally public so release binaries can be downloaded without granting access to the private source repository.

Development, source history, CI internals, issues, and unreleased code remain in the private source repository. This repository should stay limited to distribution metadata and release assets.
