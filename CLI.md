# Loadward CLI

This is the public command reference for the Loadward binary. The installed binary's `--help` output is authoritative for the exact build you are running.

Start with:

```bash
loadward --help
loadward help <command>
```

## Daemon and configuration

Running `loadward` without a command starts the daemon.

```text
loadward [--config PATH] [--check]
```

- `--config PATH` selects a configuration file.
- `--check` validates configuration and exits without starting the daemon.

The normal configuration path is `~/.config/loadward/config.toml`.

```bash
loadward --check
loadward --config /path/to/config.toml --check
```

## Direct execution

### `run`

Run one workload in an environment matching declared requirements.

```text
loadward run [placement options] [--working-dir DIR] -- COMMAND [ARG...]
```

Placement defaults are deliberately safe:

```text
isolation = isolated
workspace = none
location  = local when otherwise equivalent
```

Requirements:

- `--isolation isolated|container|vm|host`
- `--workspace none|read-only|read-write|host`
- `--desktop-session`
- `--gpu`

Preference:

- `--prefer-location local|cloud`

Host execution must be requested explicitly with both `--isolation host --workspace host`.

Examples:

```bash
loadward run -- /bin/sh -c 'echo ok'
loadward run --desktop-session -- ./scripts/desktop-proof
loadward run --gpu --prefer-location cloud -- ./scripts/gpu-proof
loadward run --workspace read-only --working-dir /workspace/project -- ./scripts/ci
```

Requests are matched against authorized compatible execution pools. A direct request does not queue inside Loadward waiting for capacity: if no compatible pool can admit immediately, the request fails without acquiring a pool slot.

### `explain`

Explain the same placement decision without reserving capacity or starting a resource.

```text
loadward explain [placement options] [--json]
```

Examples:

```bash
loadward explain
loadward explain --desktop-session
loadward explain --gpu --prefer-location cloud
loadward explain --desktop-session --json
```

Human output shows normalized requirements, the current would-select candidate, ranked compatible pools with admission pressure, and rejected pools with reasons. The result is a snapshot, not a reservation.

## Fleet status and diagnosis

### `status`

`status` is fast and read-only. Bare status is **fleet-wide**.

```text
loadward status [--target TARGET] [--route ROUTE]
loadward status --summary
loadward status --all
```

Examples:

```bash
loadward status
loadward status --target my-project
loadward status --route host
loadward status --target my-project --route desktop
loadward status --summary
loadward status --summary --json
loadward status --all
```

- `--target` and/or `--route` focus full status.
- `--summary` is a compact fleet-wide summary and cannot be combined with selectors or `--all`.
- `--all` is the explicit full-fleet form and cannot be combined with selectors.

### `doctor`

`doctor` performs deeper read-only repository-route, backend-prerequisite, and runtime-ownership checks.

Unlike `status`, bare `loadward doctor` is deliberately rejected. Diagnosis requires explicit scope:

```text
loadward doctor [--target TARGET] [--route ROUTE]
loadward doctor --all
```

Examples:

```bash
loadward doctor --target my-project
loadward doctor --route host
loadward doctor --target my-project --route desktop
loadward doctor --all
```

Use `--all` only when you intentionally want an installation-wide deep diagnosis.

## GitHub issue reporting

### `report`

Create an issue on a discovered repository target, or reuse an exact existing report when a deduplication key matches.

```text
loadward report --target TARGET --title TITLE (--body TEXT | --body-file FILE) [--key KEY]
```

Example:

```bash
loadward report \
  --target my-project \
  --title 'Validation failed' \
  --body-file report.md
```

## Conservative cleanup

### `cleanup`

Remove stale resources only when Loadward can prove ownership and inactivity conservatively.

```text
loadward cleanup [--target TARGET] [--route ROUTE] [--dry-run]
```

Inspect the plan first:

```bash
loadward cleanup --dry-run --target my-project
```

## Disruptive reset

### `reset`

Reset is non-destructive by default: without `--apply`, it only prints the plan.

```text
loadward reset [--target TARGET] [--route ROUTE]
loadward reset --apply --target TARGET --route ROUTE
loadward reset --apply --all
```

Applying reset requires either an exact target/route pair or explicit `--all`.

```bash
loadward reset --target my-project --route desktop
loadward reset --apply --target my-project --route desktop
```

## Runtime drain and resume

### `drain`

Globally stop new admission while allowing already-owned work to converge to idle.

```text
loadward drain
```

### `resume`

Resume admission after a runtime drain.

```text
loadward resume
```

If durable maintenance mode is enabled, use `loadward maintenance exit` instead.

## Durable maintenance

Maintenance mode survives daemon restarts and is the correct boundary for planned binary replacement.

```text
loadward maintenance enter
loadward maintenance status [--json]
loadward maintenance exit
```

- `enter` persists a unique maintenance epoch before draining.
- a daemon starting while the marker exists comes up drained and does not start GitHub listeners.
- `exit` explicitly activates the current daemon revision and clears the durable marker only after successful resume.

## Authorization

Permanent authorization policy is configured by trusted principal kind in TOML. Temporary grants are exact principal-kind + principal-id + gate tuples held in daemon memory and expire automatically.

```text
loadward authorization list [--json]
loadward authorization grant --kind KIND --id ID --gate GATE --ttl DURATION
loadward authorization revoke --kind KIND --id ID --gate GATE
```

Gates are:

- `host`
- `workspace-read`
- `workspace-write`
- `cloud`
- `gpu`
- `desktop-session`

The daemon's normal trusted principal kinds are:

- `local-control` with ID `local-user` for the local control socket;
- `github` with the exact repository identity, for example `incirci/my-project`.

Examples:

```bash
loadward authorization list

loadward authorization grant \
  --kind local-control \
  --id local-user \
  --gate host \
  --ttl 30m

loadward authorization revoke \
  --kind local-control \
  --id local-user \
  --gate host
```

Authorization gates **new admission**. Expiry or revocation does not retroactively terminate already-admitted durable work.

## Fail-closed recovery

Normal recovery is automatic. These operator commands are for states where Loadward cannot prove external reality strongly enough to release durable ownership or reopen a pool automatically.

```text
loadward recovery status [--json]
loadward recovery resolve --execution EXECUTION_ID
loadward recovery confirm --pool POOL
```

`recovery status` is read-only.

`resolve` and `confirm` are trusted operator mutations and are accepted only while Loadward is drained:

- `resolve` terminalizes one preparing Execution with no persisted backend identity only after authoritative external verification proves no resource was created.
- `confirm` reopens one incomplete-recovery pool only after external reality is verified; Loadward still checks every remaining durable backend identity before opening it.

Do not use these commands to bypass uncertain ownership.

## Build identity

### `version`

```text
loadward version [--json]
```

It reports the version, exact source commit, and modified-state provenance embedded by the canonical build.

## Help

```text
loadward --help
loadward help
loadward help <command>
loadward <command> --help
```

The command-specific help from the installed binary should be preferred when this document and a differently versioned binary disagree.

## JSON and automation

Commands that support `--json` emit machine-readable output for scripts and acceptance checks. Human tables are intended for operators.

For normal installation and GitHub configuration, see [`SETUP.md`](SETUP.md). For the GitHub issue-ingress path used by ChatGPT, see [`CHATGPT.md`](CHATGPT.md).
