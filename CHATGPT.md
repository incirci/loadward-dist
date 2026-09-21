# Using Loadward with ChatGPT through GitHub issue ingress

This guide covers the GitHub issue-ingress path. ChatGPT uses its connected GitHub account to create a durable execution request; Loadward separately consumes that issue through its GitHub App and dispatches the canonical execution workflow.

```text
ChatGPT
   |
connected GitHub account
   |
[loadward exec] <route> issue
   |
Loadward daemon-native issue ingress
   |
.github/workflows/exec.yml
   |
GitHub Actions: runs-on <route>
   |
Loadward route -> internal profile -> provider
```

No public Loadward endpoint, inbound daemon port, ChatGPT token, OpenAI API key, or Loadward private key is required.

Source-built deployments may also expose `loadward-mcp` for local agents or environments that support the required MCP actions. MCP is an alternative ingress to the same Loadward placement/admission/provider core.

## 1. Complete normal GitHub setup first

Follow [`SETUP.md`](SETUP.md) and prove an ordinary route workflow works:

```bash
loadward --check
loadward status
loadward doctor --target loadward
```

## 2. Use the `loadward` repository as the control repository

The daemon-native issue ingress uses the **installed repository whose basename is `loadward`** as the single control repository.

The GitHub App installation must include that repository. Repository scope comes from the App installation; there is no control-repository alias in current TOML configuration.

The issue ingress accepts requests only when the issue author is the owner of the control repository.

## 3. GitHub App permissions

The Loadward GitHub App must have:

```text
Actions:        Read and write
Administration: Read and write
Issues:         Read and write
```

After changing permissions, approve the updated installation access and restart Loadward.

## 4. Configure routes ChatGPT may request

Routes are global across the GitHub App installation. A safe local-container route is:

```toml
[github.routes.local-isolated]
isolation = "container"
workspace = "none"
location = "local"
```

A trusted host route is intentionally much stronger:

```toml
[github.routes.host]
isolation = "host"
workspace = "host"
```

There is no per-repository route allowlist. Repositories in the same App installation see the same configured routes.

## 5. Add the canonical execution workflow

The `loadward` control repository must contain `.github/workflows/exec.yml`.

Copy [`examples/exec.yml`](examples/exec.yml). It accepts exactly:

```text
route
request_id
task_b64
```

and uses the route directly as the single runner label:

```yaml
runs-on: ${{ inputs.route }}
```

Unknown route names simply have no matching Loadward scale set. The workflow never maps route names to internal profiles.

## 6. Connect GitHub to ChatGPT

Connect the GitHub account that owns the control repository using ChatGPT's GitHub app/plugin connection.

This authorization is separate from the GitHub App used by Loadward:

```text
ChatGPT -> your GitHub user authorization
Loadward -> your Loadward GitHub App installation
```

Do not give ChatGPT the Loadward GitHub App private key.

## 7. Test with a harmless request

Ask ChatGPT to create one issue in the `loadward` repository:

```text
Title:
[loadward exec] local-isolated

Body:
task_b64=<base64-encoded Bash task>
```

For example, encode:

```bash
printf 'hello from ChatGPT through Loadward\n'
```

The body must contain exactly one `task_b64=` field with no explanatory text.

With a GitHub connection that permits issue creation, ChatGPT can perform this step automatically; the user does not need to run `gh issue create`.

## 8. What Loadward does

For each valid owner-created request the daemon:

1. validates the requested route against configured global routes;
2. validates the single `task_b64` field;
3. derives `issue-<number>` as the request ID;
4. reconciles whether that exact request already has a workflow run;
5. respects current route capacity before dispatch;
6. dispatches `.github/workflows/exec.yml` on `main`;
7. rewrites the issue body with the workflow run URL/ID;
8. closes the issue.

The issue is durable. If Loadward is unavailable, it remains open until the daemon can process it.

## 9. Inspect the result

ChatGPT can read the closed issue, follow the run link, inspect the job logs, and report the result.

Locally:

```bash
loadward doctor --target loadward
journalctl --user -u loadward.service -f
```

## 10. Choose routes deliberately

Prefer isolated routes for work that does not need host access.

- `local-isolated`: disposable local container.
- `workspace-ro` / `workspace-rw`: controlled host workspace access, when configured.
- `desktop`: disposable graphical VM, when configured.
- `host`: direct execution as the daemon user; highly privileged.
- cloud/GPU routes: remote or accelerator-backed execution, when configured.

These are GitHub-facing route names. Internal profile names such as `isolated-local`, `desktop-local`, and `host-local` are not part of the issue protocol.

## Security rules

Treat `task_b64` as executable code.

- Only owner-created requests are accepted.
- Only configured routes are accepted.
- GitHub App installation membership is the repository trust boundary.
- Do not put credentials, tokens, private keys, or secrets in `task_b64`; GitHub retains issue and workflow history.
- Do not expose the Loadward control socket or GitHub App private key to ChatGPT.
- Configure privileged routes only when their trust boundary is intentional.

## Protocol reference

```text
Title:
[loadward exec] <route>

Body:
task_b64=<base64-encoded-bash>
```

Loadward rejects malformed owner requests rather than guessing compatibility variants.
