# Using Loadward with ChatGPT through GitHub issue ingress

This guide covers Loadward's narrow GitHub issue-ingress path for genuinely ad-hoc work. For normal repository development, the repository being changed should own its own workflow and use a Loadward route directly.

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
route requirements
   |
authorization + placement + admission
   |
execution pool -> backend -> execution resource
```

No public Loadward endpoint, inbound daemon port, ChatGPT token, OpenAI API key, or Loadward private key is required.

## 1. Complete normal GitHub setup first

Follow [`SETUP.md`](SETUP.md) and prove an ordinary route workflow works:

```bash
loadward --check
loadward status --summary
loadward doctor --target loadward
```

## 2. Configure the control repository explicitly

Daemon-native issue ingress is opt-in. Configure the exact installed repository that will accept execution issues:

```toml
[github]
exec_issue_repository = "incirci/loadward"
```

The value must be the full `owner/repository` identity and the repository must belong to the GitHub App installation. When this field is absent, issue ingress is disabled; Loadward does not guess a control repository from a basename such as `loadward`.

The issue ingress accepts requests only when the issue author is the owner of the configured control repository.

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

Route availability is authorization-aware. For privileged contracts, `authorization.github` may allow, deny, or require an exact temporary grant for the repository principal. A route name does not bypass authorization.

For example, with:

```toml
[authorization.github]
host = "require-grant"
```

grant host admission temporarily to the exact control repository:

```bash
loadward authorization grant \
  --kind github \
  --id incirci/loadward \
  --gate host \
  --ttl 30m
```

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

Unknown route names have no matching Loadward scale set. The workflow never maps route names to pool or backend identities.

## 6. Connect GitHub to ChatGPT

Connect the GitHub account that owns the control repository using ChatGPT's GitHub connection.

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

## 8. What Loadward does

For each valid owner-created request the daemon:

1. validates the requested route;
2. validates the single `task_b64` field;
3. derives `issue-<number>` as the request ID;
4. reconciles whether that exact request already has a workflow run;
5. projects route availability through current authorization and pool admission;
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

Prefer the narrowest environment that satisfies the task.

- isolated routes: disposable containers for ordinary work;
- workspace routes: controlled host-checkout access;
- desktop: disposable graphical VM;
- host: direct execution as the daemon user;
- cloud/GPU routes: remote or accelerator-backed execution.

These are GitHub-facing route names. Pool names remain internal operational identities.

## Security rules

Treat `task_b64` as executable code.

- Only owner-created requests are accepted.
- Only configured routes are accepted.
- GitHub App installation membership establishes the trusted repository set.
- Privileged admission is still governed by authorization policy and exact grants.
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
