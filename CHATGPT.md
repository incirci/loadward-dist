# Using RunMeSome with ChatGPT

ChatGPT does not connect directly to the RunMeSome daemon. The supported path uses GitHub as the control plane:

```text
ChatGPT
   |
   | connected GitHub account
   | creates one execution issue
   v
GitHub control repository
   |
   | RunMeSome GitHub App polls the issue
   v
RunMeSome daemon
   |
   | dispatches .github/workflows/exec.yml
   v
GitHub Actions queue
   |
   | exact RunMeSome profile
   v
RunMeSome execution provider
```

This means the daemon needs no public HTTP endpoint, inbound port, ChatGPT token, or OpenAI API key. ChatGPT talks to GitHub; RunMeSome separately talks to GitHub through the GitHub App configured in [`SETUP.md`](SETUP.md).

## 1. Complete the normal RunMeSome setup first

Follow [`SETUP.md`](SETUP.md) and make sure an ordinary workflow can already run successfully through RunMeSome.

Do not debug ChatGPT integration until this works:

```bash
runmesome -check
runmesome doctor
```

and the repository smoke workflow completes on a RunMeSome profile.

## 2. Choose a control repository

RunMeSome currently uses exactly one repository as the issue-based ChatGPT/control ingress. In the RunMeSome configuration, that target must have the literal target name:

```toml
name = "runmesome"
```

The GitHub repository itself can have any repository name. For example:

```toml
[[targets]]
name = "runmesome"
url = "https://github.com/alice/my-runmesome-control"
profiles = ["isolated-local"]
```

The target name selects the control role; the URL selects the actual repository.

For a first setup, use a dedicated private repository that you own, for example `my-runmesome-control`.

### Current owner-only restriction

The issue ingress accepts execution requests only when the GitHub user that created the issue is the owner of the control repository. Therefore the control repository should currently be owned directly by the same GitHub user account that you connect to ChatGPT.

An organization-owned control repository is not suitable for this owner-only ingress because an issue created by a human member is created by that user, not by the organization account.

## 3. Install the RunMeSome GitHub App on the control repository

The GitHub App configured for the daemon must be installed on the control repository and must have:

```text
Actions:        Read and write
Administration: Read and write
Issues:         Read and write
```

If you add the control repository or change permissions after the app was already installed, approve the updated installation access in GitHub and restart RunMeSome after changing `config.toml`.

Validate the control target:

```bash
runmesome -check
systemctl --user restart runmesome.service
runmesome doctor --target runmesome
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

This ChatGPT GitHub connection is separate from the GitHub App used by the RunMeSome daemon:

```text
ChatGPT -> your GitHub user authorization
RunMeSome -> your RunMeSome GitHub App installation
```

Do not give ChatGPT the RunMeSome GitHub App private key.

## 6. Test ChatGPT with a harmless task

Ask ChatGPT something like:

```text
Use my connected GitHub account.

In OWNER/CONTROL_REPOSITORY, submit a RunMeSome execution request using the
isolated-local profile.

Execute exactly this Bash task:

printf 'hello from ChatGPT through RunMeSome\n'

Use the RunMeSome issue protocol:
- issue title: [runmesome exec] isolated-local
- issue body: exactly one line, task_b64=<base64-encoded Bash task>
- add no other body text

After creating the issue, give me the issue link.
```

Replace `OWNER/CONTROL_REPOSITORY` with the real repository, for example:

```text
alice/my-runmesome-control
```

ChatGPT should create one issue whose title is exactly:

```text
[runmesome exec] isolated-local
```

and whose body is exactly:

```text
task_b64=<BASE64_VALUE>
```

There must be no Markdown fence, explanation, second field, or additional body text.

## 7. What RunMeSome does with the issue

The daemon polls the control repository directly through its GitHub App connection.

For each valid owner-created request it:

1. validates the requested profile against the profiles configured on the `runmesome` target;
2. validates the single `task_b64` field;
3. derives `issue-<number>` as the request ID;
4. checks whether that exact request was already dispatched;
5. dispatches `.github/workflows/exec.yml` on `main` when needed;
6. records the GitHub Actions run URL on the issue;
7. closes the issue.

If the daemon is temporarily unavailable, the issue remains open and acts as the durable request. No runner is needed merely to hand the request from ChatGPT to RunMeSome.

## 8. Check the result from ChatGPT

After submitting a request, you can ask ChatGPT:

```text
Check the RunMeSome issue you just created and tell me whether it was dispatched.
If it contains a GitHub Actions run link, inspect that run and summarize the result.
```

Or inspect locally:

```bash
runmesome doctor --target runmesome
journalctl --user -u runmesome.service -f
```

The normal control flow is:

```text
ChatGPT creates issue
    -> daemon sees issue
    -> daemon dispatches exec.yml
    -> issue is updated with run URL and closed
    -> GitHub job queues
    -> RunMeSome provides the requested execution environment
    -> job completes
    -> execution environment is cleaned up
```

## 9. Choose profiles deliberately

Start with:

```text
isolated-local
```

It runs the task in a disposable Docker environment and is the safest simple smoke test.

Other profiles intentionally expose different capabilities. For example, a workspace or direct-host profile may let a task operate on code already present on the daemon host. `host-local` is especially powerful because the Bash task executes with the permissions of the Linux user running RunMeSome.

Only expose profiles on the control target that you intentionally want ChatGPT-triggered requests to be able to use:

```toml
[[targets]]
name = "runmesome"
url = "https://github.com/alice/my-runmesome-control"
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
- Do not expose the RunMeSome control socket or GitHub App private key to ChatGPT.
- Do not expose a high-privilege profile such as `host-local` unless you intentionally trust ChatGPT-triggered tasks with that boundary.
- Review GitHub and ChatGPT account permissions independently.

## Protocol reference

A valid execution issue has this exact shape:

```text
Title:
[runmesome exec] <profile>

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

RunMeSome rejects malformed owner requests instead of trying to guess or accept compatibility variants.
