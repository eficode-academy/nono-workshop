# Credential Injection

## Learning Goals

- Understand why raw secrets shouldn't be handed directly to an AI agent
- Store a credential in your system's keystore (macOS Keychain, Linux Secret Service)
- Inject a credential as an environment variable with `--env-credential`
- Inject a credential via nono's local reverse proxy with `--credential`, so the agent never sees the raw secret
- Add credentials to a profile so they apply automatically every run
- Verify a credential is actually hidden from the sandboxed process

## Introduction

An AI agent that can read its own environment variables can read your API keys — and an agent that's been prompt-injected or is simply buggy can leak them, intentionally or not. nono gives you two ways to hand credentials to a sandboxed process without that risk:

- **Environment variable injection** (`--env-credential`) — loads a secret from your system keystore and sets it as an environment variable *before* the sandbox is applied. Simple, but the secret is visible to the process if it inspects its own environment.
- **Proxy injection** (`--credential`, recommended for API keys) — the agent talks to a local reverse proxy on `127.0.0.1`. The proxy injects the real key into the outbound HTTP request as it forwards it upstream. The agent never has the real key in its environment or memory — only a placeholder ("phantom token").

```
Agent sends:  POST http://127.0.0.1:PORT/openai/v1/chat/completions
Proxy sends:  POST https://api.openai.com/v1/chat/completions
              Authorization: ****** (injected from keystore)
```

Both approaches read secrets from the same place: your OS keystore, or optionally 1Password, Bitwarden, or Apple Passwords via `op://`, `bw://`, and `apple-password://` URIs.

<details>
<summary>:bulb: Which one should I use?</summary>

Use **proxy injection** for anything that's an HTTP API call — LLM provider keys (OpenAI, Anthropic, Gemini), GitHub/GitLab tokens, internal APIs. Use **environment injection** only for secrets that aren't consumed over HTTP (e.g. a database password read directly by a library) — there's no proxy to intercept those.

</details>

## Exercise

### Overview

- Store a fake test credential in your system's keystore
- Inject it as an environment variable and confirm the sandboxed process can see it
- Inject an LLM-style credential through the proxy and confirm the sandboxed process only sees a placeholder
- Add both patterns to a profile so they apply automatically
- Use `nono why` to reason about what's exposed

### Step by step instructions

<details>
<summary>Step by step</summary>

#### Part 1: Store a credential in your keystore

**Task 1: macOS — store a test secret in Keychain**

```bash
security add-generic-password -s "nono" -a "workshop-demo" -w "super-secret-value"
```

**Task 1 (alt): Linux / WSL2 — store a test secret via Secret Service**

```bash
echo -n "super-secret-value" | secret-tool store --label="nono: workshop-demo" \
    service nono username workshop-demo target default
```

> :bulb: On Linux/WSL2 the attribute names matter: use `username` (not `account`), and always include `target default` — the keyring crate nono uses requires both to find the entry. If `secret-tool` isn't installed, get it from the `libsecret-tools` (Debian/Ubuntu) or equivalent package, and make sure a Secret Service provider like `gnome-keyring` is running.

#### Part 2: Environment variable injection

**Task 2: Run a command with the secret injected as an env var**

```bash
nono run --allow-cwd --env-credential workshop-demo -- env
```

Look for `WORKSHOP_DEMO` (or similar) in the output — the raw secret value is visible in the child process's environment, exactly as if you'd exported it yourself. This confirms the mechanism works, but also shows *why* it's less safe than proxy injection: anything the process does with its environment (logging it, echoing it, passing it to a subprocess) can leak the raw value.

#### Part 3: Proxy injection (recommended for API keys)

**Task 3: Store a fake LLM API key**

macOS:

```bash
security add-generic-password -s "nono" -a "openai" -w "sk-fake-workshop-key"
```

Linux/WSL2:

```bash
echo -n "sk-fake-workshop-key" | secret-tool store --label="nono: openai" \
    service nono username openai target default
```

**Task 4: Run with the credential proxy enabled**

```bash
nono run --allow-cwd --network-profile claude-code --credential openai -- env
```

Look at the environment this time — you'll see `OPENAI_BASE_URL=http://127.0.0.1:<port>/openai` set for you, but **no `OPENAI_API_KEY` anywhere**. Any HTTP client inside the sandbox that respects `OPENAI_BASE_URL` (most LLM SDKs do) gets redirected through the local proxy, which injects the real key only on the way out to `api.openai.com`.

> :bulb: On WSL2, proxy-only network mode is blocked by default due to a `seccomp` conflict with WSL2's interop process, and must be explicitly force-enabled per-profile (`wsl2_proxy_policy: "insecure_proxy"`). Prefer environment injection or run credential-proxied workloads on native Linux/macOS where possible.

**Task 5: Confirm with `nono why`**

```bash
nono why --self --path ~/.aws --op read --json
```

This is the same mechanism an agent itself can call when it hits a denied operation, to understand *why* and how to request access properly, instead of guessing or retrying blindly.

#### Part 4: Put it in a profile

Wiring credentials into a profile means every future run gets them automatically — no need to remember flags.

**Task 6: Add environment credential injection to a profile**

```jsonc
{
  "meta": { "name": "my-agent" },
  "env_credentials": {
    "workshop-demo": "WORKSHOP_DEMO_TOKEN"
  }
}
```

**Task 7: Add proxy credential injection to a profile**

```jsonc
{
  "meta": { "name": "my-agent-secure" },
  "network": {
    "custom_credentials": {
      "openai": {
        "upstream": "https://api.openai.com/v1",
        "credential_key": "openai",
        "env_var": "OPENAI_API_KEY",
        "inject_header": "Authorization",
        "credential_format": "******"
      }
    },
    "credentials": ["openai"]
  }
}
```

```bash
nono run --profile my-agent-secure -- my-agent
```

Your agent's environment now contains a phantom token like `OPENAI_API_KEY=nono_sess_a1b2c3...` — never the real key. Only the proxy, running outside the sandbox, swaps in the real value when forwarding the request to `api.openai.com`.

</details>

### Clean up

Remove the test credentials you created:

```bash
# macOS
security delete-generic-password -s "nono" -a "workshop-demo"
security delete-generic-password -s "nono" -a "openai"

# Linux/WSL2
secret-tool clear service nono username workshop-demo target default
secret-tool clear service nono username openai target default
```

## Summary

You stored secrets in your OS keystore instead of a `.env` file, injected one directly as an environment variable, and injected another through nono's credential proxy so the raw value never entered the sandbox at all — only a phantom token did. You then wired both patterns into a profile so they apply automatically on every run.

Head over to the [next exercise](03-audit-and-rollback.md) to review a tamper-evident record of everything your agent did, and learn how to undo any filesystem changes it made.
