---
title: "My Dotfiles: A DevOps-Focused Development Environment"
date: 2026-03-31
draft: false
tags: ["dotfiles", "devops", "chezmoi", "fish", "neovim", "tooling", "mise"]
description: "How I manage my dotfiles with chezmoi and the opinionated toolchain I've settled on after years of iteration."
ShowToc: true
---

Managing a development environment across machines is a solved problem — if you're willing to invest a little upfront. This post walks through how I manage my dotfiles using [chezmoi](https://www.chezmoi.io/) and the opinionated toolchain I've settled on after years of iteration.

---

## Why chezmoi?

Dotfiles managers come in many flavors: bare git repos, GNU Stow, custom shell scripts. I landed on chezmoi for a few reasons:

- **Templating**: A single `dot_gitconfig.tmpl` can render different values per machine (work email vs. personal email) using Go templates.
- **No symlinks**: chezmoi copies files to their destination, so you're never one `rm` away from a broken config.
- **First-class secrets support**: integrates with password managers, age encryption, and environment variables.
- **Single binary**: no Ruby, no Python, no system dependencies — just one binary managed by mise.

The naming convention is simple: `dot_foo` becomes `~/.foo`, `dot_config/bar/` becomes `~/.config/bar/`. Template files end in `.tmpl` and are rendered before deployment.

---

## Bootstrapping a new machine

Everything starts with one script:

```bash
bash debian-startup.sh
```

This script handles:

- Installing base system packages (`curl`, `git`, `ripgrep`, `fd-find`, etc.)
- Setting up Docker
- Installing Nerd Font (FiraCode)
- Installing mise (the runtime version manager)
- Cloning and applying the LazyVim starter for Neovim
- Setting Fish as the default shell

From zero to productive in one command.

---

## The toolchain

### mise: one tool to rule them all

Instead of managing Node via `nvm`, Python via `pyenv`, and so on, I use [mise](https://mise.jdx.dev/). A single `config.toml` pins versions for everything:

- Runtimes: Node 24
- Kubernetes/DevOps: kubectl, helm, k9s, jq, yq, stern
- Editor: neovim (latest)
- Productivity: lazygit, lazydocker, yazi
- Utilities: ast-grep, chezmoi, tree-sitter, usage

`mise install` and you're done. No more "works on my machine" for tooling.

### Fish shell + pure

Fish gives me autosuggestions and syntax highlighting out of the box. I lean heavily on `abbr` (abbreviations) rather than aliases — they expand inline so you always see the full command before it runs.

A few favorites:

```fish
abbr -a --set-cursor='%' -- gp 'git add -A && git commit -m "%" && git push'
abbr -a -- dr 'docker run --rm -it'
abbr -a -- kg 'kubectl get -n'
abbr -a --set-cursor='%' -- kr 'kubectl run -it --rm --restart=Never --image=% -- sh'
```

The `--set-cursor` flag is a Fish-specific trick that positions the cursor mid-expansion — useful for commit messages where you always end up typing in the same spot.

I replaced the default prompt with [pure](https://github.com/pure-fish/pure) — minimal by default, and every bit of behavior is a `pure_*` universal variable rather than a config file. I track `fish_variables` in the dotfiles repo too.

### Neovim (LazyVim)

I use [LazyVim](https://www.lazyvim.org/) as a base and add a small set of plugins on top:

- **bufferline** — tab-like buffer navigation with Shift+arrows
- **smart-splits** — pane navigation with Alt+arrows
- **nvim-spider** — smarter `w`/`b`/`e` motions that respect camelCase and underscores
- **snacks** — fuzzy picker configured to include hidden files by default
- **tokyonight** — theme in transparent mode so the terminal background shows through

### WezTerm

[WezTerm](https://wezfurlong.org/wezterm/) is my terminal emulator of choice — GPU-accelerated, cross-platform, configured in Lua.

Notable config:

- FiraCode Nerd Font for ligatures and icons
- `Ctrl+Shift+|` / `Ctrl+Shift+?` to split panes horizontally/vertically
- `Ctrl+Shift+<` / `Ctrl+Shift+>` to rotate panes
- `Ctrl+Shift+{` / `Ctrl+Shift+}` to reorder tabs

### Claude Code

[Claude Code](https://claude.ai/code) is Anthropic's CLI for Claude and a permanent fixture in my workflow — it sits in the right pane of the `dev` zellij tab. I keep custom subagent definitions in `dot_claude/agents/`, which deploy to `~/.claude/agents/`. Each agent is a markdown file with YAML frontmatter declaring its name, model, and allowed tools, followed by a focused system prompt:

- **Coder** (Sonnet) — writes Terraform, Kubernetes manifests, Bash scripts, and Python tooling. Knows the conventions cold so I don't have to repeat them.
- **Planner** (Opus) — researches the codebase and docs, identifies risks, produces an ordered implementation plan. Never touches code.
- **Security Reviewer** (Opus) — reviews infrastructure diffs for IAM overpermissioning, exposed secrets, RBAC misconfigs, and container security issues. Reports only, never fixes.

The split between Planner and Coder mirrors the think-then-do discipline I try to apply manually — except now it's enforced by separate system prompts and model choices.

Alongside the agents, `dot_claude/skills/` holds skills that gate infra changes on specific tool invocations rather than relying on manual review steps:

- `tf-fmt-validate`, `tf-plan-diff`, `cost-estimate` — Terraform format/validate, plan diff, and infracost estimate
- `k8s-dry-run`, `rbac-diff` — Kubernetes/Helm server-side dry run and RBAC before/after diff
- `policy-scan`, `secrets-scan` — tfsec/kube-score static scanning, gitleaks secret scanning
- `shellcheck-gate` — shellcheck on any bash script I write or edit
- `provider-docs-lookup` — fetch pinned-version docs for a Terraform provider resource, Kubernetes API object, or Helm chart

They deploy to `~/.claude/skills/` the same way the agents do. Each one expects its underlying tool (`terraform`, `infracost`, `tfsec`, `kube-score`, `gitleaks`, `shellcheck`) on `PATH` via mise.

### Zellij

Zellij is the session layer. I set up `Ctrl+b` prefix mode (matching tmux muscle memory, built into Zellij) and a larger scrollback. The `devops.sh` script attaches to (or creates) a named session using the `devops` layout, a two-tab setup:

- **dev**: Neovim on the left, Claude on the right
- **ops**: k9s on top, a shell below

Each pane launches its app directly as a `command` pane — no `remain-on-exit` config needed, Zellij already keeps the pane open when the command exits and re-runs it on `ENTER`. A `devops` fish function wraps `zellij attach --create <session> --default-layout devops`, keying the session name off the current directory when `$KUBECONFIG` is set.

---

## Workflow

Day-to-day the workflow is:

```bash
# Edit a config in the source directory
chezmoi edit config.fish

# Preview what will change
chezmoi diff

# Apply to home directory
chezmoi apply

# Commit and push
chezmoi git -- add -A && chezmoi git -- commit -m "..." && chezmoi git -- push
```

On a new machine:

```bash
mise x chezmoi -- chezmoi init --apply thewalterman
```

That's it. Full environment in minutes.

---

## Conclusion

A good dotfiles setup is invisible when it works — you sit down at a new machine and it just feels like home. chezmoi handles the mechanics, mise pins the tooling, and Fish + pure keep the shell fast and informative. The upfront investment is a few hours; the payoff is every machine, forever.

If you're curious, the full setup is on GitHub at [thewalterman/dotfiles](https://github.com/thewalterman/dotfiles).
