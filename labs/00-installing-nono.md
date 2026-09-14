# Installing Nono

## Learning Goals

- Understand what nono is and how it sandboxes AI agents
- Install nono on macOS or Linux
- Set up WSL2 on Windows and install nono inside it
- Verify your installation and check sandbox support on your machine
- Run your first sandboxed command with `nono run`

## Introduction

**nono** is a sandboxing runtime for AI agents. When you run a command through nono, it is wrapped in an OS-level sandbox that restricts what files, network hosts, and credentials that command (and everything it spawns) can access — enforced directly by the kernel, not by a container or VM.

nono uses a different mechanism depending on your OS:

| Platform | Mechanism |
|----------|-----------|
| Linux (kernel 5.13+) | Landlock LSM |
| macOS | Seatbelt (`sandbox_init`) |
| Windows (WSL2) | Landlock LSM, via the WSL2 kernel |
| Windows (native) | Not supported |

<details>
<summary>:bulb: Why doesn't nono support native Windows?</summary>

nono relies on Landlock and Seatbelt — kernel sandboxing primitives that only exist on Linux and macOS. Windows has no equivalent primitive that nono can hook into directly, which is why Windows users run nono inside WSL2: a real Linux kernel running alongside Windows.

</details>

Once a sandbox is applied, it cannot be removed or widened from inside — not even by nono itself. That's what makes it safe to hand an AI agent the keys, without handing it your entire filesystem.

## Exercise

### Overview

- Install nono using the method for your OS (macOS, Linux, or WSL2 on Windows)
- Verify the CLI is on your `PATH`
- Run `nono setup --check-only` to confirm sandbox support
- Run your first sandboxed command with `nono run --allow .`
- Confirm the sandbox actually blocks access outside the allowed directory

### Step by step instructions

<details>
<summary>Step by step</summary>

#### Part 1: Install nono

Pick the path that matches your machine.

**Task 1a: macOS or Linux — Homebrew (recommended)**

```bash
brew install nono
```

**Task 1b: macOS or Linux — install script**

If you don't use Homebrew:

```bash
curl -fsSL https://nono.sh/install.sh | sh
```

<details>
<summary>:bulb: Other Linux package managers</summary>

- **Debian/Ubuntu (.deb):**

  ```bash
  VERSION=$(curl -sIL https://github.com/nolabs-ai/nono/releases/latest | grep -i location | grep -oP 'v\K[0-9a-zA-Z.-]+')
  ARCH=$(dpkg --print-architecture)
  wget https://github.com/nolabs-ai/nono/releases/download/v${VERSION}/nono-cli_${VERSION}_${ARCH}.deb
  sudo dpkg -i nono-cli_${VERSION}_${ARCH}.deb
  ```

- **Fedora (COPR):**

  ```bash
  sudo dnf install 'dnf-command(copr)'
  sudo dnf copr enable always-further/nono
  sudo dnf install nono-cli
  ```

- **RHEL / openSUSE (manual RPM):**

  ```bash
  VERSION=$(curl -sIL https://github.com/nolabs-ai/nono/releases/latest | grep -i location | grep -oP 'v\K[0-9a-zA-Z.-]+')
  ARCH=$(uname -m)
  wget https://github.com/nolabs-ai/nono/releases/download/v${VERSION}/nono-cli-${VERSION}-1.${ARCH}.rpm
  sudo dnf install ./nono-cli-${VERSION}-1.${ARCH}.rpm   # or: sudo zypper install ./...
  ```

- **Arch Linux (AUR):**

  ```bash
  yay -S nono-ai-bin   # or: paru -S nono-ai-bin
  ```

- **Nix:**

  ```bash
  nix shell nixpkgs#nono
  ```

Full details are in the [installation docs](https://nono.sh/docs/cli/getting_started/installation).

</details>

**Task 1c: Windows — set up WSL2, then install inside it**

nono does not run natively on Windows — you need WSL2 first.

1. Open **PowerShell as Administrator** and install WSL2 with a Linux distribution (Ubuntu is the default and recommended for this workshop):

   ```powershell
   wsl --install
   ```

   > :bulb: If WSL is already installed, make sure you're on WSL**2** (not WSL1): run `wsl -l -v` and check the `VERSION` column. Upgrade a distro with `wsl --set-version <DistroName> 2`.

2. Restart your computer if prompted, then launch your new Linux distro from the Start menu. On first launch you'll be asked to create a Linux username and password — this is separate from your Windows login.

3. Inside the WSL2 terminal, check your kernel version — nono needs kernel 5.13+ for Landlock, and WSL2's kernel (6.6 by default) already ships with it enabled:

   ```bash
   uname -r
   ```

4. Update packages and install nono using the Linux instructions from Task 1b above (Homebrew on Linux also works inside WSL2):

   ```bash
   sudo apt update
   curl -fsSL https://nono.sh/install.sh | sh
   ```

> :bulb: From here on, **do all your work inside the WSL2 terminal**, not PowerShell or cmd.exe. VS Code's [Remote - WSL extension](https://code.visualstudio.com/docs/remote/wsl) lets you edit WSL2 files from the regular Windows VS Code UI if you prefer a GUI editor.

<details>
<summary>:bulb: What works differently on WSL2?</summary>

WSL2 runs a real Linux kernel, so core filesystem sandboxing, sensitive-path blocking, and dangerous-command blocking all work exactly like native Linux. Two advanced features are currently limited by the Microsoft-built kernel:

- **Per-port TCP filtering** needs Landlock ABI V4 (kernel 6.7+); WSL2 ships 6.6, so only allow/block-all network works, not fine-grained per-port rules
- **Credential proxy injection** is blocked by default on WSL2 because of a `seccomp` conflict with WSL2's own interop process — it can be force-enabled per-profile, but fails secure by default

Everything else — profiles, `nono run`, `nono shell`, audit, rollback, GPU passthrough — works the same as native Linux. See the [WSL2 support docs](https://nono.sh/docs/cli/internals/wsl2) for the full matrix.

</details>

#### Part 2: Verify your installation

**Task 2: Check the version**

```bash
nono --version
```

Expected output (version may vary):

```
nono 0.4.2
```

> :bulb: If you get "command not found", make sure the install location is on your `PATH` — restart your terminal, or on Linux/WSL2 re-source your shell profile (`source ~/.bashrc`).

**Task 3: Check sandbox support**

```bash
nono setup --check-only
```

Expected output includes a feature availability report, e.g.:

```
Testing sandbox support...
  Filesystem sandbox (Landlock/Seatbelt) ... available
  Sensitive path blocking ................ available
  Dangerous command blocking ............. available
  Block-all network ...................... available
```

On WSL2, this same command prints the WSL2 feature matrix instead, showing which advanced features (like per-port filtering) aren't available on your kernel yet.

#### Part 3: Run your first sandbox

**Task 4: Create a scratch directory**

```bash
mkdir ~/nono-workshop-scratch
cd ~/nono-workshop-scratch
echo "hello from inside the sandbox" > allowed.txt
```

**Task 5: Run a command sandboxed to only this directory**

```bash
nono run --allow . -- cat allowed.txt
```

Expected output:

```
hello from inside the sandbox
```

**Task 6: Confirm the sandbox actually blocks access elsewhere**

Try to read a file outside the allowed directory, like your SSH keys:

```bash
nono run --allow . -- cat ~/.ssh/id_rsa
```

Expected output (a permission error from the kernel, not from `cat` itself):

```
cat: /home/you/.ssh/id_rsa: Permission denied
```

> :bulb: This isn't `cat` politely refusing — the kernel itself is refusing to open the file for this process. Confirm why with `nono why`:
>
> ```bash
> nono why --path ~/.ssh/id_rsa --op read
> # DENIED - sensitive_path (SSH keys and config)
> ```

</details>

### Clean up

Remove the scratch directory:

```bash
rm -rf ~/nono-workshop-scratch
```

## Summary

You installed nono on your platform of choice — macOS, Linux, or WSL2 on Windows — verified it can enforce a kernel-level sandbox, and ran your first command with restricted filesystem access. You also saw firsthand that sensitive paths like `~/.ssh` are blocked by default, even without any configuration.

Head over to the [next exercise](01-setting-up-your-profile.md) to set up a profile tailored to your AI agent.
