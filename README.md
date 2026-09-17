# RunMeSome Distribution

Private binary distribution repository for RunMeSome.

This repository intentionally contains no RunMeSome source code. Authorized users receive packaged release binaries through **Releases** only.

## Download

You must have access to this private repository and be authenticated with GitHub.

### Browser

Open **Releases**, choose the required version, and download the binary for your Linux architecture.

### GitHub CLI

Authenticate once:

```bash
gh auth login
```

Download the latest x86-64 build and checksums:

```bash
mkdir -p /tmp/runmesome-install
cd /tmp/runmesome-install

gh release download \
  --repo incirci/runmesome-dist \
  --pattern runmesome-linux-amd64 \
  --pattern SHA256SUMS

grep ' runmesome-linux-amd64$' SHA256SUMS | sha256sum -c -
chmod +x runmesome-linux-amd64
install -Dm755 runmesome-linux-amd64 ~/.local/bin/runmesome
```

For ARM64, replace `amd64` with `arm64`.

## Releases

Releases are created only by an explicitly triggered release workflow in the private RunMeSome source repository. Ordinary commits and merges do not publish releases.

Each release contains:

- `runmesome-linux-amd64`
- `runmesome-linux-arm64`
- `SHA256SUMS`
- `BUILDINFO.txt`

`BUILDINFO.txt` records the release version and exact private-source commit used for the build.

## Access model

This repository must remain private.

For true download-only access, host this repository under a GitHub organization and grant users the **Read** repository role. Private repositories owned by a personal GitHub account cannot grant collaborators read-only access; personal-repository collaborators receive write access.
