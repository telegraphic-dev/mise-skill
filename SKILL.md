---
name: mise
description: Use when managing development tools, project environments, or task commands with mise. Prefer reproducible mise.toml workflows and safe, non-interactive agent execution.
version: 1.0.0
author: Telegraphic Developer
license: MIT
metadata:
  hermes:
    tags: [mise, dev-tools, tool-versions, environments, tasks, reproducibility]
    related_skills: []
---

# mise

## Overview

[mise](https://mise.jdx.dev/) is a single CLI for project tool versions, environment variables, and tasks. A committed `mise.toml` makes the same runtime setup usable in a shell, editor, CI, and an agent session.

For agents, the important distinction is simple:

- **Use `mise exec` and `mise run`** to execute inside a project environment without changing the calling shell.
- **Use shell activation only for interactive human shells.** It is not required for automation and is a lousy default in a one-shot agent command.
- Treat project configuration as code: inspect it before trusting it, and avoid silently rewriting it.

Authoritative docs: <https://mise.jdx.dev/>

## When to Use

- A repository has `mise.toml`, `.mise.toml`, or a legacy `.tool-versions` file.
- A project needs pinned Node.js, Python, Java, Go, Bun, Terraform, or another development tool.
- A project exposes build, test, lint, or release commands through mise tasks.
- A project needs reproducible environment variables, `.env` loading, or CI setup.
- You need to diagnose why a command uses the wrong runtime version.

Do **not** use this skill merely to update mise itself. Verify the installed version and follow the upstream release guidance; changing a shared developer tool is an environment change, not routine project work.

## Installation

Install the **mise binary** before using any workflow below. Installing this skill only provides guidance; it does not install mise. Verify first:

```sh
mise --version
```

### macOS and Linux — recommended upstream binary

The official `mise.run` installer downloads a prebuilt binary and installs it to `~/.local/bin/mise` by default. It is the upstream-recommended method for macOS and Linux; use the documented GPG verification path when independent installer verification is required.

```sh
# Inspect the installer first; do not blindly pipe remote code into a shell.
curl -fsSL https://mise.run -o /tmp/mise-install.sh
less /tmp/mise-install.sh
sh /tmp/mise-install.sh

# The binary may not yet be on PATH.
~/.local/bin/mise --version
```

To install to a deliberate system path instead, use `MISE_INSTALL_PATH` and obtain approval for the privileged write:

```sh
curl -fsSL https://mise.run | MISE_INSTALL_PATH=/usr/local/bin/mise sh
mise --version
```

For a user who explicitly wants installation **and** shell activation in one step, upstream provides shell-specific installers:

```sh
curl https://mise.run/bash | sh  # installs and updates ~/.bashrc
curl https://mise.run/zsh | sh   # installs and updates ~/.zshrc
curl https://mise.run/fish | sh  # installs and updates Fish config
```

These modify shell configuration. Do not run them for an agent, CI job, or another user without explicit approval.

### Package-manager alternatives

Use the platform-native option when system package management is required. Package manager releases can lag the official binary.

```sh
# macOS (upstream prefers mise.run over Homebrew)
brew install mise

# Ubuntu 26.04+
sudo add-apt-repository -y ppa:jdxcode/mise
sudo apt update && sudo apt install -y mise

# Debian 11+ / Ubuntu 22.04+
sudo apt install -y extrepo
sudo extrepo enable mise
sudo apt update && sudo apt install -y mise

# Alpine / Arch
apk add mise
sudo pacman -S mise

# Windows PowerShell: Scoop is upstream-recommended; winget is also supported.
scoop install mise
winget install jdx.mise
```

For Fedora/RHEL, Nix, Snap, Cargo, npm-distributed binary, Docker, and other platforms, use the authoritative installation matrix: <https://mise.jdx.dev/installing-mise.html>

### Verify and activate for a human shell

Activation is optional. `mise exec` and `mise run` work without it and are preferred for agents, scripts, and CI.

```sh
# Verify the binary before editing a shell profile.
mise --version
mise doctor

# Bash
printf '\neval "$(mise activate bash)"\n' >> ~/.bashrc

# Zsh
printf '\neval "$(mise activate zsh)"\n' >> "${ZDOTDIR-$HOME}/.zshrc"

# Fish
printf '\nmise activate fish | source\n' >> ~/.config/fish/config.fish
```

Start a fresh shell, then verify `mise doctor`, `which mise`, and `mise --version`. For PowerShell, use the exact profile command in the upstream installation documentation; profile locations vary.

### 1. Detect and inspect configuration

From the repository root, first identify the configuration mise will load:

```sh
mise config ls
mise ls --current
mise tasks ls
```

Then read each project config before executing tasks or trusting it. Look particularly for `[env]`, `env` files, `[tasks]`, hooks, and `includes`; configuration can run commands or load secrets.

Do not run `mise trust` blindly. Trusting a config authorizes it for mise, including its hooks. Ask for approval when a previously untrusted configuration must be trusted or when it contains behavior you cannot explain.

### 2. Use the project environment without shell activation

Prefer these forms in agents, scripts, CI, and remote commands:

```sh
# Run one command with tools resolved from the project config.
mise exec -- node --version
mise exec -- python --version

# Run a declared project task.
mise run test
mise run lint

# Explicitly select a tool for one command without modifying mise.toml.
mise exec node@24 -- node --version
```

`mise exec` installs missing declared tools when necessary. That is an external download and can consume time, bandwidth, or disk; report it when it happens rather than pretending it was instant.

## Core Workflows

### Add or change a project tool

Use a clear version request, then review the resulting `mise.toml` diff:

```sh
mise use node@24
mise use python@3.13
mise use java@21
mise install
mise ls --current
```

For a personal default rather than repository configuration:

```sh
mise use --global node@24
```

Do not casually use `--global` in a project task. It changes the machine environment and does not make the project reproducible.

A major/minor request such as `node@24` follows that release series. Use exact pins and a lockfile when every machine and CI run must resolve the same artifact:

```sh
mise lock
mise install --locked
```

Read the lockfile documentation before enabling `--locked` globally: <https://mise.jdx.dev/dev-tools/mise-lock.html>

### Run project tasks

Discover tasks before guessing task names:

```sh
mise tasks ls
mise tasks info test
mise tasks validate
```

Then run the declared task:

```sh
mise run test
mise run build
# `mise test` is shorthand only when a task named test exists.
mise test
```

Tasks inherit mise-managed tools and configured environment variables. Prefer task commands over manually reconstructing a project's build command; that is the point of having a task runner in the first place.

### Manage environment variables

A minimal, reviewable configuration:

```toml
[tools]
node = "24"
python = "3.13"

[env]
NODE_ENV = "development"
_.file = ".env"

[tasks.test]
description = "Run the test suite"
run = "npm test"
```

Inspect `.env` handling carefully. Never commit production credentials to `mise.toml`, `.env`, or a lockfile. Use the project’s secret store/CI secrets and keep local secret files ignored.

To see the resolved environment while debugging:

```sh
mise env
mise env node@24
```

Use `mise --no-env` if you need to isolate a problem from mise-provided variables.

### Shell activation for interactive users

Activation makes selected tool binaries and environment variables appear automatically on `cd`. It is useful in a human shell, optional everywhere else.

```sh
# bash
printf '\neval "$(mise activate bash)"\n' >> ~/.bashrc

# zsh
printf '\neval "$(mise activate zsh)"\n' >> ~/.zshrc

# fish
mise activate fish | source
```

Do not edit shell profiles without the user’s approval. After a user has enabled activation, verify in a fresh shell:

```sh
mise doctor
which node
node --version
```

### CI and automation

CI should install mise, then run a project task or explicit command. It should not rely on shell activation.

```sh
mise install --yes
mise run test
# or
mise exec -- npm ci && mise exec -- npm test
```

Pin the mise action or installer version according to the organization’s supply-chain policy. Review upstream CI guidance: <https://mise.jdx.dev/continuous-integration.html>

Use `mise install --locked` only after committing a verified lockfile. The locked mode intentionally fails when required URLs are absent, which is a feature, not a bug wearing a fake moustache.

## Safe Agent Defaults

1. Start in the intended repository directory (`mise -C /path/to/repo ...` when needed).
2. Read `mise.toml` and run `mise config ls`, `mise ls --current`, and `mise tasks ls` before modifying anything.
3. Prefer `mise exec -- <command>` and `mise run <task>` over activation or global defaults.
4. Use `mise use` only when the user asked to change project tool policy; inspect and commit the config/lockfile changes together.
5. Never run `mise trust` or a task from an unreviewed repository without approval.
6. Never put credentials in `mise.toml` or source-controlled `.env` files.
7. Before diagnosing tooling, check the selected runtime with `mise which <binary>` and `mise ls --current`.

## Troubleshooting

### `node`, `python`, or another binary is the wrong version

```sh
mise config ls
mise ls --current
mise which node
mise exec -- node --version
```

If `mise exec -- node --version` is correct but `node --version` is wrong, mise itself is fine; shell activation or `PATH` ordering is wrong. Run `mise doctor` and inspect the shell initialization.

### A project config is not being discovered

```sh
mise config ls
mise -C /absolute/path/to/project config ls
mise doctor path --full
```

Run commands from the repository root. Nested configs and environment-specific config files can override parent settings; `mise config ls` shows the actual load order.

### A task fails only in CI or an agent

```sh
mise tasks info <task>
mise env
mise run -v <task>
```

Compare the resolved config, tool versions, working directory, and required secrets. Do not "fix" a missing secret by committing it to a config file.

### A config is blocked as untrusted

Read the file and its includes/hooks first. Then request approval before:

```sh
mise trust /path/to/mise.toml
```

### Installation/download problems

```sh
mise doctor
mise cache path
mise install -v
mise ls-remote <tool>
```

Check network/proxy policy, platform support, and the requested tool/backend. Use `mise registry <tool>` to confirm a registry entry where relevant.

## Useful Command Reference

```sh
mise --version                 # installed mise version
mise config ls                 # loaded configuration files
mise ls --current              # tools selected for this directory
mise install                   # install configured tools
mise use tool@version          # write/update project tool selection
mise exec -- command args      # execute in resolved mise environment
mise run task                  # run declared task
mise tasks ls                  # list available tasks
mise tasks validate            # validate task definitions
mise env                       # print resolved environment
mise which binary              # show selected executable path
mise doctor                    # diagnose paths and setup
mise lock                      # generate/update lockfile
mise trust config-file         # authorize a reviewed config
```

Full CLI reference: <https://mise.jdx.dev/cli/>

## Common Pitfalls

1. **Activating mise just to run a CI command.** Use `mise exec` or `mise run`; activation mutates the shell setup and is unnecessary.
2. **Using `--global` for project tooling.** It creates machine-specific behavior instead of a committed project contract.
3. **Guessing task names.** Start with `mise tasks ls`; task conventions differ by repository.
4. **Trusting configs on sight.** A config may include hooks, environment files, or tasks with side effects. Read it first.
5. **Committing secrets in `[env]`.** `mise.toml` is source code, not a vault.
6. **Ignoring config precedence.** Debug with `mise config ls`, not vibes.
7. **Assuming an installed tool is selected.** `mise ls` and `mise ls --current` answer different questions; use the latter for the active project.
8. **Installing through curl without review.** Prefer a package manager where appropriate; otherwise inspect the upstream installer and get approval before executing it.

## Verification Checklist

- [ ] `mise --version` succeeds.
- [ ] `mise config ls` shows the intended configuration files.
- [ ] `mise ls --current` shows the intended active tool versions.
- [ ] `mise tasks ls` was reviewed before selecting a task.
- [ ] `mise exec -- <tool> --version` succeeds for the affected runtime.
- [ ] `mise tasks validate` succeeds after task changes.
- [ ] Configuration and lockfile changes are reviewed, tested, and committed together.
- [ ] No credentials were added to source-controlled config or output.
