# Setting Up Your Profile

## Learning Goals

- Understand what a nono profile is and why AI agents need one
- Search the nono registry and pull a pre-built profile for your agent
- Run your agent inside that profile's sandbox
- Scaffold a brand-new profile from scratch with `nono profile init`
- Extend a base profile instead of writing every rule by hand
- Validate and inspect the profile you built

## Introduction

Typing out `--allow`, `--block-net`, and every other flag by hand every time you launch your agent gets old fast. A **profile** is a JSON(C) file that bundles all of that up — filesystem scope, network rules, credentials, hooks — under one name you pass with `--profile`.

Two ways to get a profile:

1. **Pull one from the registry.** The community and agent vendors publish signed, ready-to-use profiles for popular tools (Claude Code, GitHub Copilot CLI, OpenCode, and more) at [registry.nono.sh](https://registry.nono.sh). This is the fastest path and what most people should start with.
2. **Author your own.** Either scaffold an empty profile and fill it in, or `extend` a registry profile and only override what you need.

Profiles that extend a base **inherit everything** from it — you only need to specify what's different. List-type fields (allowed paths, denied commands) are additive; single-value fields (like `network.block`) are overridden by the child profile.

<details>
<summary>:bulb: Why not just always use --allow flags?</summary>

Flags are great for one-off experiments. Profiles are what you check into your dotfiles or share with your team, so everyone gets the same least-privilege sandbox without memorizing flags. They're also the only way to configure some features, like tool-level micro-sandboxing and credential proxies.

</details>

## Exercise

### Overview

- Search the registry for a profile matching your AI agent
- Pull it and run your agent through it
- Scaffold a new profile from scratch with `nono profile init`
- Extend a base profile to add a directory your agent needs
- Validate the resulting profile and inspect what it resolves to

### Step by step instructions

<details>
<summary>Step by step</summary>

#### Part 1: Pull a pre-built profile for your agent

**Task 1: Search the registry**

Replace `claude` with the name of whatever agent you use (`copilot`, `opencode`, `pi`, etc.):

```bash
nono search claude
```

Expected output (namespace/versions may vary):

```
nolabs-ai/claude	-	Official Claude Code Plugin
```

**Task 2: Pull the profile**

```bash
nono pull nolabs-ai/claude
```

This downloads the pack, verifies its Sigstore signature, pins the signer identity, and installs it into your local pack store at `~/.config/nono/packages/`.

> :bulb: Pin to an exact version with `nono pull nolabs-ai/claude@1.2.0` if you want reproducible installs across a team.

**Task 3: Run your agent through the profile**

```bash
nono run --profile nolabs-ai/claude -- claude
```

Your agent now runs with read/write access to the current directory and nothing else — no SSH keys, no cloud credentials, no access to the rest of your disk.

**Task 4: List what's installed**

```bash
nono list --installed
```

#### Part 2: Scaffold a profile from scratch

Registry profiles won't always cover everything — maybe your agent needs access to a `data/` directory alongside your project, or you're building a profile for an internal tool with no registry entry.

**Task 5: Generate a minimal skeleton**

```bash
nono profile init my-agent
```

Expected output: a new file at `~/.config/nono/profiles/my-agent.json` containing something like:

```jsonc
{
  "extends": "default",
  "meta": {
    "name": "my-agent",
    "description": "Profile for my agent"
  },
  "groups": {
    "include": ["deny_credentials"]
  },
  "workdir": {
    "access": "readwrite"
  },
  "filesystem": {
    "allow": [],
    "read": []
  }
}
```

> :bulb: Every profile automatically inherits from the built-in `default` profile, even if you don't write `"extends": "default"` yourself — it's already applied.

**Task 6: Generate a full skeleton to see every available section**

```bash
nono profile init my-agent-full --full --description "Full example profile"
```

Open it in your editor and skim the sections: `filesystem`, `network`, `env_credentials`, `hooks`, `rollback`, `diagnostics`. You'll fill most of these in over the next exercises — for now, just see what's there.

**Task 7: Edit your minimal profile to add a directory**

Open `~/.config/nono/profiles/my-agent.json` and add a path your agent needs, e.g. a shared data folder:

```jsonc
{
  "extends": "default",
  "meta": {
    "name": "my-agent",
    "description": "Profile for my agent"
  },
  "groups": {
    "include": ["deny_credentials"]
  },
  "workdir": {
    "access": "readwrite"
  },
  "filesystem": {
    "allow": ["~/data"],
    "read": []
  }
}
```

#### Part 3: Extend a base profile instead

Rather than starting from an empty skeleton, you can build directly on top of a registry profile and only override the parts you care about.

**Task 8: Scaffold a profile that extends a pulled profile**

```bash
nono profile init my-claude --extends nolabs-ai/claude --description "Custom Claude profile with extra tooling access"
```

**Task 9: Add your override**

Edit `~/.config/nono/profiles/my-claude.json` so it looks like:

```jsonc
{
  "extends": "nolabs-ai/claude",
  "meta": {
    "name": "my-claude",
    "description": "Custom Claude profile with extra tooling access"
  },
  "filesystem": {
    "allow": ["/opt/my-tools"],
    "read": ["/etc/my-app"]
  }
}
```

This profile inherits everything from `nolabs-ai/claude` — network rules, hooks, credentials — and only adds the two extra paths. The base profile itself is untouched.

**Task 10: Validate the profile**

```bash
nono profile validate ~/.config/nono/profiles/my-claude.json
```

**Task 11: Inspect what it actually resolves to**

Because of inheritance, the profile you wrote and the profile nono actually applies can look quite different. Check both:

```bash
# Show the fully resolved profile after all merging
nono profile show my-claude

# See exactly what changed compared to the default profile
nono profile diff default my-claude
```

**Task 12: Run your agent with the new profile**

```bash
nono run --profile my-claude -- claude
```

</details>

### Clean up

Remove the scratch profiles you created, if you don't want to keep them:

```bash
rm ~/.config/nono/profiles/my-agent.json
rm ~/.config/nono/profiles/my-agent-full.json
rm ~/.config/nono/profiles/my-claude.json
```

## Summary

You pulled a signed, ready-to-use profile from the nono registry and ran your agent through it, then scaffolded your own profile from scratch, extended it from a base profile with `extends`, and confirmed exactly what it resolves to with `nono profile show` and `nono profile diff`. You now have a repeatable, shareable way to sandbox your agent, instead of retyping flags every time.

Head over to the [next exercise](02-credential-injection.md) to give your agent access to API keys and tokens it can use — but never see.
