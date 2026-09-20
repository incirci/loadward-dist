# Using Loadward with ChatGPT through GitHub issue ingress

This guide covers the GitHub issue-ingress path available to binary-only Loadward deployments. ChatGPT does not connect directly to the daemon in this mode; GitHub acts as the durable control plane.

Source-built deployments also expose a thin `loadward-mcp` adapter that talks to the same Loadward control socket and placement/admission core. That MCP path is separate from this public binary-only guide; both paths preserve the same execution-profile trust boundaries.

```text
ChatGPT
   |
   | connected GitHub account
   | creates one execution issue
   v
GitHub control repository
   |
   | Loadward GitHub App polls the issue
   v
Loadward daemon
   |
   | dispatches .github/workflows/exec.yml
   v
GitHub Actions queue
   |
   | exact Loadward profile
   v
Loadward execution provider
```

This means the daemon needs no public HTTP endpoint, inbound port, ChatGPT token, or OpenAI API key. ChatGPT talks to GitHub; Loadward separately talks to GitHub through the GitHub App configured in [`SETUP.md`](SETUP.md).

## 1. Complete the normal Loadward setup first

Follow [`SETUP.md`](SETUP.md) and make sure an ordinary workflow can already run successfully through Loadward.

Do not debug ChatGPT integration until this works:

```bash
loadward -check
loadward doctor
```

and the repository smoke workflow completes on a Loadward profile.

## 2. Choose a control repository

Loadward currently uses exactly one repository as the issue-based ChatGPT/control ingress. In the Loadward configuration, that target must have the literal target name:

```toml
name = "loadward"
```

The GitHub repository itself can have any repository name. For example:

```toml
[[targets]]
name = "loadward"
url = "https://github.com/alice/my-loadward-control"
profiles = ["isolated-local"]
```

The target name selects the control role; the URL selects the actual repository.

For a first setup, use a dedicated private repository that you own, for example `my-loadward-control`.

### Current owner-only restriction

The issue ingress accepts execution requests only when the GitHub user that created the issue is the owner of the control repository. Therefore the control repository should currently be owned directly by the same GitHub user account that you connect to ChatGPT.

An organization-owned control repository is not suitable for this owner-only ingress because an issue created by a human member is created by that user, not by the organization account.

## 3. Install the Loadward GitHub App on the control repository

The GitHub App configured for the daemon must be installed on the control repository and must have:

```text
Actions:        Read and write
Administration: Read and write
Issues:         Read and write
```

If you add the control repository or change permissions after the app was already installed, approve the updated installation access in GitHub and restart Loadward after changing `config.toml`.

Validate the control target:

```bash
loadward -check
systemctl --user restart loadward.service
loadward doctor --target loadward
```

## 4. Add the canonical execution workflow

The control repository must contain:

```text
.github/workflows/exec.yml
```

Copy [`examples/exec.yml`](examples/exec.yml) into that path.

The workflow accepts exactly three dispatch inputs:

```text
profile
request_id
task_b64
```

`profile` is resolved through an explicit allowlist before it becomes the single `runs-on` label. The task payload cannot choose an arbitrary runner label.

Commit and push the workflow to the control repository's `main` branch.

## 5. Connect GitHub to ChatGPT

In ChatGPT, open **Apps** or **Plugins** (the label can vary by account), select **GitHub**, and connect the GitHub account that owns the control repository.

OpenAI's current connection instructions are documented here:

- https://help.openai.com/en/articles/11145903-connecting-github-to-chatgpt
- https://help.openai.com/en/articles/20001494-connecting-and-managing-app-accounts-in-chatgpt

The connected GitHub account must be able to create issues in the control repository. Depending on the ChatGPT surface or workspace policy, ChatGPT may ask for approval before performing a GitHub write action.

This ChatGPT GitHub connection is separate from the GitHub App used by the Loadward daemon:

```text
ChatGPT -> your GitHub user authorization
Loadward -> your Loadward GitHub App installation
```

Do not give ChatGPT the Loadward GitHub App private key.

## 6. Test ChatGPT with a harmless task

Ask ChatGPT something like:

```text
Use my connected GitHub account.

In OWNER/CONTROL_REPOSITORY, submit a Loadward execution request using the
isolated-local profile.

Execute exactly this Bash task:

printf 'hello from ChatGPT through Loadward\n'

Use the Loadward issue protocol:
- issue title: [loadward exec] isolated-local
- issue body: exactly one line, task_b64=<base64-encoded Bash task>
- add no other body text

After creating the issue, give me the issue link.
```

Replace `OWNER/CONTROL_REPOSITORY` with the real repository, for example:

```text
alice/my-loadward-control
```

ChatGPT should create one issue whose title is exactly:

```text
[loadward exec] isolated-local
```

and whose body is exactly:

```text
task_b64=<BASE64_VALUE>
```

There must be no Markdown fence, explanation, second field, or additional body text.

## 7. What Loadward does with the issue

The daemon polls the control repository directly through its GitHub App connection.

For each valid owner-created request it:

1. validates the requested profile against the profiles configured on the `loadward` target;
2. validates the single `task_b64` field;
3. derives `issue-<number>` as the request ID;
4. checks whether that exact request was already dispatched;
5. dispatches `.github/workflows/exec.yml` on `main` when needed;
6. records the GitHub Actions run URL on the issue;
7. closes the issue.

If the daemon is temporarily unavailable, the issue remains open and acts as the durable request. No runner is needed merely to hand the request from ChatGPT to Loadward.

## 8. Check the result from ChatGPT

After submitting a request, you can ask ChatGPT:

```text
Check the Loadward issue you just created and tell me whether it was dispatched.
If it contains a GitHub Actions run link, inspect that run and summarize the result.
```

Or inspect locally:

```bash
loadward doctor --target loadward
journalctl --user -u loadward.service -f
```

The normal control flow is:

```text
ChatGPT creates issue
    -> daemon sees issue
    -> daemon dispatches exec.yml
    -> issue is updated with run URL and closed
    -> GitHub job queues
    -> Loadward provides the requested execution environment
    -> job completes
    -> execution environment is cleaned up
```

## 9. Choose profiles deliberately

Start with:

```text
isolated-local
```

It runs the task in a disposable Docker environment and is the safest simple smoke test.

Other profiles intentionally expose different capabilities. For example, a workspace or direct-host profile may let a task operate on code already present on the daemon host. `host-local` is especially powerful because the Bash task executes with the permissions of the Linux user running Loadward.

Only expose profiles on the control target that you intentionally want ChatGPT-triggered requests to be able to use:

```toml
[[targets]]
name = "loadward"
url = "https://github.com/alice/my-loadward-control"
profiles = ["isolated-local"]
```

Do not add a broader profile merely as a fallback.

## 10. Repository code access is separate from execution access

An `isolated-local` task starts in a disposable execution environment. It does not automatically contain a checkout of some other repository and it does not inherit your daemon host's filesystem.

If a ChatGPT-triggered task needs source code, deliberately choose how that code becomes available. Examples include:

- clone public source inside the task;
- use a configured workspace execution profile for a controlled host workspace;
- use `host-local` for trusted tasks that intentionally need the daemon user's existing checkout and host permissions.

Do not weaken isolation just to make a test pass.

## 11. Security rules

Treat the issue payload as executable code.

- Only the repository owner is accepted by the current issue ingress.
- Only profiles explicitly listed on the control target can be requested.
- Do not put credentials, tokens, private keys, or other secrets inside `task_b64`; GitHub retains issue and workflow history.
- Do not expose the Loadward control socket or GitHub App private key to ChatGPT.
- Do not expose a high-privilege profile such as `host-local` unless you intentionally trust ChatGPT-triggered tasks with that boundary.
- Review GitHub and ChatGPT account permissions independently.

## Protocol reference

A valid execution issue has this exact shape:

```text
Title:
[loadward exec] <profile>

Body:
task_b64=<base64-encoded-bash>
```

Example Bash payload:

```bash
uname -a
id
```

The body must still contain only the encoded form:

```text
task_b64=dW5hbWUgLWEKaWQK
```

Loadward rejects malformed owner requests instead of trying to guess or accept compatibility variants.
