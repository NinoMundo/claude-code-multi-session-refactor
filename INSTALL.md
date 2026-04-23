# Installation

These slash commands work inside [Claude Code](https://www.anthropic.com/claude/claude-code). You pick one of two install locations depending on whether you want them available everywhere or just in one project.

## User-level install (applies to all projects)

Commands in `~/.claude/commands/` are available in every project.

```sh
mkdir -p ~/.claude/commands
cp commands/*.md ~/.claude/commands/
cp -r commands/l10n ~/.claude/commands/
```

After this, `/sprint-kickoff`, `/sprint-session-plan`, `/sprint-session-execute`, `/sprint-merge`, `/l10n-sprint-kickoff`, and `/l10n-session-plan` are usable in any Claude Code session.

## Project-level install (one repo only)

Commands in `.claude/commands/` inside a repo are available only when Claude Code runs against that repo. Useful if you want sprint commands for one codebase without installing them globally.

```sh
cd your-project
mkdir -p .claude/commands
cp /path/to/this/repo/commands/*.md .claude/commands/
cp -r /path/to/this/repo/commands/l10n .claude/commands/
```

Commit `.claude/commands/` to the repo if you want other contributors on the project to have the same commands.

## Verifying the install

Inside Claude Code, type `/` — the commands should appear in the autocomplete list. If they don't, check the file paths and that the `.md` files are readable.

## Updating

This repo is the source of truth. Pull the latest and re-copy:

```sh
git pull origin main
cp commands/*.md ~/.claude/commands/
cp -r commands/l10n ~/.claude/commands/
```

Or, if you cloned this repo to a stable location, symlink instead of copy so updates are automatic:

```sh
ln -sf /path/to/this/repo/commands/sprint-kickoff.md ~/.claude/commands/sprint-kickoff.md
# (repeat for each command)
```

## Prerequisites

- Claude Code CLI installed and authenticated
- A git repo you have write access to
- GitHub CLI (`gh`) if you want `/sprint-merge` to open and merge PRs for you
- Optional but strongly recommended: [`act`](https://github.com/nektos/act) for local CI validation before pushing (the `/sprint-session-execute` command references an `/act-run` helper that runs the GitHub Actions workflow locally via Docker)
