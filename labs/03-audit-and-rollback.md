# Audit & Rollback

## Learning Goals

- Understand what nono records for every sandboxed session by default
- List, filter, and inspect past sessions with `nono audit`
- Verify that a session's recorded audit log hasn't been tampered with
- Enable filesystem snapshots with `--rollback` to capture what an agent changed
- Review a session's file changes and restore them to their pre-run state
- Clean up old audit and rollback data to reclaim disk space

## Introduction

Sandboxing stops an agent from touching what it shouldn't — but agents still make legitimate changes to files you *did* allow them to touch, and you'll want a record of what happened and a way to undo it if something goes wrong.

nono separates this into two layers, each opt-in beyond the baseline:

| Layer | Question it answers | Enabled by |
|-------|----------------------|-----------|
| **Session audit** (default) | What command ran, when, and what did the supervisor observe? | Always on, unless `--no-audit` |
| **Audit-log integrity** (default) | Has the recorded audit log been modified since it was written? | Always on, unless `--no-audit-integrity` |
| **Filesystem integrity** | What filesystem state existed before/after the run? | `--audit-integrity` |
| **Rollback** | Can this session's file changes be restored later? | `--rollback` |

Every `nono run` is recorded into an append-only event log by a trusted supervisor process that stays outside the sandbox for the whole session. That's what makes the record trustworthy — the sandboxed agent itself never gets to write or edit its own audit trail.

<details>
<summary>:bulb: Why is rollback separate from audit?</summary>

Audit is cheap — it's just an event log and, optionally, a filesystem hash. Rollback is heavier: it stores full content-addressable snapshots of every tracked file, before and after the run, so you can actually restore them. Not every session needs that, so it's opt-in via `--rollback`.

</details>

## Exercise

### Overview

- Run your agent normally and inspect the audit trail it left behind
- Filter and search past sessions with `nono audit list`
- Verify a session's integrity with `nono audit verify`
- Re-run with `--rollback` enabled and let your agent make a change
- Review the diff and restore the file to its original state
- Clean up old sessions with `nono audit cleanup` and `nono rollback cleanup`

### Step by step instructions

<details>
<summary>Step by step</summary>

#### Part 1: Every run is already audited

**Task 1: Create a scratch project and run your agent through it**

```bash
mkdir ~/nono-workshop-scratch && cd ~/nono-workshop-scratch
echo "line one" > notes.txt
nono run --allow . -- claude   # or `copilot`, or `opencode`
```

Do something small with your agent — ask it to edit `notes.txt` — then exit.

**Task 2: List your recorded sessions**

```bash
nono audit list --path ~/nono-workshop-scratch
```

Expected output (id and timestamp will differ):

```
nono 1 command(s)

  ~/nono-workshop-scratch (1 commands)
    20260219-092017-8117  just now  completed  claude
```

> :bulb: Sessions are grouped by project directory automatically. Combine filters, e.g. `nono audit list --command claude --since 2026-02-01`, to narrow down a long history.

**Task 3: Show the full detail of that session**

```bash
nono audit show 20260219-092017-8117
```

This prints the command that ran, start/end timestamps, exit code, and every audit event the supervisor recorded — including any capability decisions if your agent tried to access something outside its sandbox.

> :bulb: Add `--json` to any `audit` or `rollback` command for machine-readable output suitable for compliance tooling or log aggregation.

#### Part 2: Verify the audit log wasn't tampered with

**Task 4: Verify the session's integrity**

```bash
nono audit verify 20260219-092017-8117
```

This recomputes the hash-chain over the recorded events and checks the Merkle root nono committed at the end of the session. If anyone had edited `audit-events.ndjson` by hand after the fact, this would fail.

<details>
<summary>:bulb: Attesting a session with a signing key</summary>

For stronger guarantees — e.g. proving *who* ran a session, not just that the log is unmodified — sign the completed session once it finishes:

```bash
nono run --audit-sign-key default --allow . -- claude
```

The supervisor signs the final audit root and writes a DSSE attestation bundle into the session directory. Verify it the same way, optionally pinning to a specific public key:

```bash
nono audit verify <session-id> --public-key-file ./audit-signing-key.pub
```

</details>

#### Part 3: Snapshot and roll back filesystem changes

Audit tells you *what happened*. Rollback lets you *undo it*.

**Task 5: Run with rollback enabled**

```bash
nono run --rollback --allow . -- claude
```

Ask your agent to make an obviously destructive edit for this exercise — e.g. "delete everything in notes.txt and replace it with TODO" — then exit the agent.

Because file changes were detected, nono immediately shows an interactive review:

```
[nono] Session complete. 1 file changed:
  ~ notes.txt (modified)

Show diff for notes.txt? [y/N]
Restore notes.txt to its original state? [y/N]
```

> :bulb: Use `--no-rollback-prompt` if you're scripting this and don't want the interactive UI — snapshots are still taken silently.

**Task 6: List rollback-tracked sessions**

```bash
nono rollback list --path ~/nono-workshop-scratch
```

**Task 7: Inspect the change without restoring yet**

```bash
nono rollback show <session-id> --diff
```

This prints a familiar unified, git-style diff of exactly what the agent changed.

**Task 8: Restore the file**

If you didn't restore it during the interactive prompt, do it now:

```bash
nono rollback restore <session-id>
```

Confirm `notes.txt` is back to `line one`:

```bash
cat notes.txt
```

> :bulb: Preview a restore before committing to it with `nono rollback restore <session-id> --dry-run`.

**Task 9: Verify the snapshot store itself is intact**

```bash
nono rollback verify <session-id>
```

This recomputes the Merkle tree over the stored objects and reports any missing or corrupted data — useful after copying rollback data between machines or restoring from backup.

#### Part 4: Clean up old data

Audit and rollback data accumulate over time. Both have dedicated cleanup commands.

**Task 10: Preview and clean up audit sessions**

```bash
# Preview what would be removed
nono audit cleanup --keep 10 --dry-run

# Actually remove everything but the 10 newest sessions
nono audit cleanup --keep 10
```

**Task 11: Preview and clean up rollback snapshots**

```bash
# Preview
nono rollback cleanup --older-than 30 --dry-run

# Remove rollback snapshots older than 30 days
nono rollback cleanup --older-than 30
```

> :bulb: By default nono automatically prunes rollback storage once it exceeds 10 sessions or 5 GB — manual cleanup is for when you want to reclaim space sooner, or enforce a retention policy in CI.

</details>

### Clean up

Remove the scratch project and any leftover workshop sessions:

```bash
rm -rf ~/nono-workshop-scratch
nono audit cleanup --keep 0 --dry-run   # review first
nono rollback cleanup --all             # requires confirmation
```

## Summary

You confirmed that every `nono run` session is recorded and integrity-protected by default, learned to filter and inspect that history with `nono audit list`/`show`/`verify`, and used `--rollback` to give your agent a safety net — capturing a snapshot before and after each run so any unwanted change can be reviewed and reversed. Together with the sandboxing and credential-injection exercises, you now have the full loop: restrict what an agent can do, hide the secrets it uses, and keep a tamper-evident, reversible record of everything it actually did.
