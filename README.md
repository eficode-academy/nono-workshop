# Nono Workshop

This workshop will take you from "what is nono?" to running your AI coding agent inside a kernel-enforced sandbox, with credentials it can never see.

It's going to be a lot of fun!

## Prerequisites

- A Mac, Linux machine, or Windows machine with [WSL2](https://learn.microsoft.com/en-us/windows/wsl/)
- A terminal (macOS Terminal, Linux shell, or a WSL2 shell on Windows)
- An AI coding agent you'd like to sandbox — e.g. [Claude Code](https://github.com/anthropics/claude-code), [GitHub Copilot CLI](https://github.com/github/copilot-cli), or [OpenCode](https://github.com/opencode-ai/opencode) (any CLI-based agent works)
- Nothing else installed yet — exercise 00 covers installing nono itself

> **Note:** nono works natively on macOS and Linux. On Windows, it runs inside WSL2 (Windows Subsystem for Linux) — there is no native Windows support, so exercise 00 includes a dedicated WSL2 setup section.

## Philosophy

This tutorial is designed to be self-paced to make the most of your time.

The exercises build on each other, so work through them in order. Each exercise has:

- **Learning Goals** — what you'll know after completing it
- **Introduction** — the minimum context you need
- **Exercise** — an overview for experienced users, with collapsible step-by-step instructions for those who want more guidance

Don't be afraid to experiment — that's how you learn!

## Exercises

| # | Exercise | Description |
|---|----------|-------------|
| 00 | [Installing Nono](labs/00-installing-nono.md) | Install nono on macOS and Linux, set up WSL2 on Windows, and run your first sandboxed command |
| 01 | [Setting Up Your Profile](labs/01-setting-up-your-profile.md) | Pull a pre-built profile for your AI agent from the registry, then scaffold and extend your own profile from scratch |
| 02 | [Credential Injection](labs/02-credential-injection.md) | Give your agent access to API keys and tokens it can use but never see, using environment and proxy injection |
| 03 | [Audit & Rollback](labs/03-audit-and-rollback.md) | Review a tamper-evident record of everything your agent did, and snapshot/restore any filesystem changes it made |

## Ready to begin?

Head over to [the first exercise](labs/00-installing-nono.md) to begin.

## Cheat Sheet

For a quick reference of the most common nono commands, see [CHEATSHEET.md](CHEATSHEET.md).

## What is nono?

[nono](https://github.com/nolabs-ai/nono) is an open-source sandboxing runtime for AI agents. It uses kernel-level primitives — Landlock on Linux, Seatbelt on macOS, and Landlock inside WSL2 on Windows — to enforce least-privilege sandboxes with zero setup, no daemon, no container, and no VM. Once applied, a sandbox cannot be widened from the inside, not even by nono itself.

## Roadmap

Future exercises we're planning:

- **Tool Sandboxing** — micro-sandboxing the tools your agent calls, like `git`, `gh`, and `curl`
- **Network Policies** — allow-listing domains and filtering API calls at Layer 7
- **Publishing Packs** — building and signing your own profile packs for the nono registry
