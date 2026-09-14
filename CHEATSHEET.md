# Nono Cheat Sheet

Quick reference for the commands used in these exercises.

## Installation

| Command | Description |
|---------|-------------|
| `brew install nono` | Install nono via Homebrew (macOS/Linux) |
| `curl -fsSL https://nono.sh/install.sh \| sh` | Install nono via the install script |
| `wsl --install` | Install WSL2 with a default Linux distro (Windows PowerShell, admin) |
| `wsl -l -v` | List installed WSL distros and their version |
| `nono --version` | Print the installed nono version |
| `nono setup --check-only` | Check what sandbox features are supported on this machine |

## Running Sandboxed Commands

| Command | Description |
|---------|-------------|
| `nono run --allow . -- <cmd>` | Run `<cmd>` with read/write access to the current directory only |
| `nono run --allow-cwd -- <cmd>` | Shorthand for allowing the current working directory |
| `nono run --profile <name> -- <cmd>` | Run `<cmd>` inside a named profile's sandbox |
| `nono run --block-net -- <cmd>` | Run with all outbound network access blocked |
| `nono run --network-profile <name> -- <cmd>` | Run with host-level network filtering via proxy |
| `nono run --allow-domain <host> -- <cmd>` | Allow a specific domain through the network proxy |
| `nono shell --allow .` | Start an interactive shell with sandbox permissions |
| `nono why --path <path> --op read` | Check if a path would be allowed or denied, and why |
| `nono why --self --path <path> --op read --json` | Query current sandbox capabilities from inside the sandbox (JSON) |

## Packs & Registry

| Command | Description |
|---------|-------------|
| `nono search <keyword>` | Search the registry for profiles matching a keyword |
| `nono pull <namespace>/<name>` | Install a pack (profile bundle) from the registry |
| `nono pull <namespace>/<name>@<version>` | Install a specific pinned version |
| `nono pull <namespace>/<name> --init` | Also copy project-level instruction files into the current directory |
| `nono list --installed` | List installed packs |
| `nono outdated` | Show which installed packs have newer versions |
| `nono update` | Update all installed packs to the latest version |
| `nono pin <namespace>/<name>` | Pin a pack so `nono update` skips it |
| `nono remove <namespace>/<name>` | Uninstall a pack |

## Authoring Profiles

| Command | Description |
|---------|-------------|
| `nono profile init <name>` | Scaffold a minimal profile skeleton |
| `nono profile init <name> --extends <base>` | Scaffold a profile that extends a base profile |
| `nono profile init <name> --full` | Scaffold a profile with every available section stubbed out |
| `nono profile init <name> --output <path>` | Write the scaffold to a specific file instead of the default profile directory |
| `nono profile validate <path>` | Validate a profile file |
| `nono profile show <name>` | Print the fully resolved profile (after all `extends` merging) |
| `nono profile diff default <name>` | Show what changed compared to the built-in default profile |
| `nono profile schema` | Print the JSON Schema for profile files (for editor autocomplete) |
| `nono profile guide` | Print the embedded LLM-oriented profile authoring guide |

## Credential Injection

| Command | Description |
|---------|-------------|
| `security add-generic-password -s "nono" -a "<name>" -w "<secret>"` | Store a credential in macOS Keychain |
| `secret-tool store --label="nono: <name>" service nono username <name> target default` | Store a credential in Linux/WSL2 Secret Service |
| `nono run --env-credential <name> -- <cmd>` | Inject a keystore credential as an environment variable |
| `nono run --env-credential-map 'op://<vault>/<item>/<field>' <ENV_VAR> -- <cmd>` | Inject a 1Password secret as a named environment variable |
| `nono run --credential <service> -- <cmd>` | Inject a credential via the local reverse proxy (recommended for API keys) |
| `nono run --network-profile <name> --credential <service> -- <cmd>` | Combine host-level network filtering with proxy credential injection |

## Profile Concepts

| Concept | Description |
|---------|-------------|
| **Profile** | A JSON(C) file bundling filesystem, network, credential, and hook rules under one name |
| **`extends`** | Inherit all rules from a base profile; list fields are additive, single-value fields are overridden |
| **`default` profile** | Automatically merged into every profile, even without an explicit `extends` |
| **Pack** | A signed bundle of profiles, hooks, and plugins distributed via the registry |
| **Phantom token** | A placeholder credential value the agent sees instead of the real secret, swapped out by the proxy |
| **Sensitive paths** | Paths like `~/.ssh`, `~/.aws`, `~/.gnupg` blocked by default regardless of profile |
| **Landlock** | The Linux kernel LSM nono uses for filesystem/network sandboxing (also used inside WSL2) |
| **Seatbelt** | The macOS sandboxing mechanism (`sandbox_init`) nono uses |

## Audit

| Command | Description |
|---------|-------------|
| `nono run --allow . -- <cmd>` | Every run is audited by default — no extra flag needed |
| `nono run --no-audit -- <cmd>` | Disable audit recording for this session |
| `nono run --no-audit-integrity -- <cmd>` | Keep the audit log but skip the tamper-evident integrity layer |
| `nono run --audit-integrity -- <cmd>` | Also hash filesystem state for tracked writable paths, pre/post run |
| `nono run --audit-sign-key <key> -- <cmd>` | Sign the completed session's audit root once, at finalization |
| `nono audit list` | List recorded sessions, grouped by project directory |
| `nono audit list --path <dir> --since <date> --command <name>` | Filter sessions by path, date range, or command |
| `nono audit show <session-id>` | Show full detail for one session, including recorded events |
| `nono audit verify <session-id>` | Verify the audit log's integrity and ledger inclusion |
| `nono audit cleanup --keep <n>` / `--older-than <days>` | Remove old audit sessions to reclaim disk space |

## Rollback

| Command | Description |
|---------|-------------|
| `nono run --rollback -- <cmd>` | Snapshot tracked files before and after the run, so changes can be reviewed and restored |
| `nono run --rollback --no-rollback-prompt -- <cmd>` | Take snapshots without the interactive post-run review (for scripting) |
| `nono rollback list` | List past rollback-tracked sessions, grouped by project |
| `nono rollback show <session-id> --diff` | Show a unified diff of what changed in a session |
| `nono rollback restore <session-id>` | Restore files from a session back to their pre-run state |
| `nono rollback restore <session-id> --dry-run` | Preview a restore without modifying files |
| `nono rollback verify <session-id>` | Verify the integrity of a stored snapshot (Merkle tree + objects) |
| `nono rollback cleanup --keep <n>` / `--older-than <days>` | Remove old rollback snapshots to reclaim disk space |
