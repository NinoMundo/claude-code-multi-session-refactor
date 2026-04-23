# claude-code-multi-session-refactor

Subagents and methodology for running multi-session refactoring sprints with [Claude Code](https://www.anthropic.com/claude/claude-code).

## What this is

A set of slash commands and a written playbook for running large refactoring sprints — work that's too big for a single pull request but too small for a rewrite. Think: localizing a multi-year codebase, migrating a dependency, rolling out a new testing framework, cleaning up a security audit's findings.

The approach breaks the work into many small sessions. Each session follows the same loop: investigate → propose a plan → approve → execute one commit at a time → gate on CI → merge. A planning Claude (you in this chat) reviews each plan before a coding Claude (Claude Code CLI) executes it. Humans sit between them as the approval layer.

The methodology was developed during a 24-session localization sprint on [Sweet Home Paraguay](https://github.com/NinoMundo/SHP-App), a Flutter + Supabase app. See [`docs/case-study.md`](docs/case-study.md) for that story.

## What's in this repo

- **`commands/`** — slash commands you install into Claude Code. Four general ones work for any sprint type; two l10n-specific extensions layer on top.
- **`docs/methodology.md`** — the long-form playbook. Reusable patterns and the reasoning behind each.
- **`docs/l10n-conventions.md`** — l10n-specific conventions (the `commonX` reuse rule, ICU plurals, brand-name discipline, humanize helpers for enum-to-label mappings).
- **`docs/example-agents-md.md`** — template for a project's `AGENTS.md` (the working document the coder reads at the start of every session).
- **`docs/case-study.md`** — narrative walkthrough of the SHP-App sprint.
- **`docs/lessons-learned.md`** — things that went wrong along the way, and the checklist items that now catch them.
- **`examples/session-plan-template.md`** — empty `SESSION_N_PLAN.md` to copy-paste for a new session.

## Quick install

Copy the `commands/` directory into your Claude Code commands folder. See [`INSTALL.md`](INSTALL.md) for user-level vs project-level installs.

```sh
# User-level (applies to all projects)
cp commands/*.md ~/.claude/commands/
cp -r commands/l10n ~/.claude/commands/

# Project-level (one repo only)
mkdir -p .claude/commands
cp commands/*.md .claude/commands/
cp -r commands/l10n .claude/commands/
```

Then read [`docs/methodology.md`](docs/methodology.md) before starting your first sprint.

## The sprint lifecycle

```
 1. /sprint-kickoff       Investigate the codebase. Propose the session split.
                          Write AGENTS.md. Set up CI gates. Approve the plan.

 2. /sprint-session-plan  For each session: new branch, read-only investigation,
                          write SESSION_N_PLAN.md, commit the plan, push,
                          wait for human approval.

 3. /sprint-session-execute  Execute approved plan. Small commits. Run gates
                             (dart analyze, tests, builds). Use /act-run
                             locally to pre-validate before pushing.

 4. /sprint-merge         Open PR, squash-merge, delete branch, pull main,
                          report the new main SHA. Repeat from step 2
                          for the next session.
```

## Why this works

Three things that are unusual about the loop:

1. **Investigation is a separate step.** Every session starts read-only. The coder writes a plan file before touching any source code. This catches scope errors, dead code, hidden exclusions, and missed opportunities before any commit lands.

2. **Reuse discipline compounds.** When a string or helper appears in two or more domains, it gets promoted to a shared location with grep-verified evidence. Over many sessions, a pool of shared keys accumulates — later sessions cash it in instead of duplicating. The 24-session SHP-App sprint ended with a single late session hitting 17 reuses in one file.

3. **Prior-session exclusions are preserved.** Each session inherits a growing list of "don't touch this, here's why." The playbook teaches the coder to read AGENTS.md first and honor every exclusion. Regressions across sessions stay rare.

## License

MIT. See [`LICENSE`](LICENSE).

## Contributing

This repo is a living playbook. Real sprint experience → documented pattern → subagent update. If you run a sprint using this methodology and learn something, a PR adding to `docs/lessons-learned.md` is welcome.
