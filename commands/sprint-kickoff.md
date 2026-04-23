---
description: Start a multi-session refactoring sprint. Investigate the codebase, propose a session split, seed AGENTS.md and CI gates.
---

You are starting a multi-session refactoring sprint on the current repository. This is a pre-session setup phase. No refactoring yet — just investigation, session planning, and tooling setup.

## Inputs (ask the user if not provided)

1. **Sprint goal in one sentence.** Examples: "Localize all user-facing strings to ARB." "Remove unused dependencies." "Migrate from package A to package B." "Address findings from the security audit report at `AUDIT.md`."
2. **Scope.** Which directories or file patterns are in scope? Which are explicitly out (generated code, vendored libraries, test fixtures)?
3. **Hard constraints.** Is there a CI workflow the work must not break? Deployment windows to respect? Features in active development that shouldn't be touched?
4. **Existing conventions.** Is there an `AGENTS.md`, `CONTRIBUTING.md`, or similar doc that captures how this project works? If yes, read it first.

## What to do

### Step 1: Read before you write

Read the following (in this order, if present):

- `README.md`
- `AGENTS.md` (if any)
- `CONTRIBUTING.md`
- `.github/workflows/*.yml` (understand existing CI)
- `package.json` / `pubspec.yaml` / `Cargo.toml` / etc. (language + deps)
- Any `AUDIT.md` or similar findings document the user mentions

Report back a short summary of the project shape: language, framework, platforms, existing CI, and any conventions the sprint needs to respect.

### Step 2: Scope investigation

Based on the sprint goal, enumerate:

- **Files in scope.** Count them. List the top 10–20 by size or relevance.
- **Total rough estimate of "units of work"** (strings to migrate, deps to remove, findings to address, etc.).
- **Prior work that has touched this area.** Git-blame or log search for commits related to the sprint topic. Avoid duplicating.
- **Dead code candidates.** Anything that looks like it could be deleted outright rather than migrated.

This is read-only. Do not modify any source files in this step.

### Step 3: Propose a session split

Break the work into sessions. Good session sizing:

- **Target one file or one tight file group per session** where possible.
- **Target ~30 minutes of human review time** per session plan. If a session would need a 2,000-line plan document, split it.
- **Sensitive work goes last.** Anything payment-related, security-critical, or schema-altering should come after the simpler sessions build confidence in the pattern.
- **Foundational work goes first.** Infrastructure (CI gates, new shared helpers, ARB scaffold) comes before domain migrations that depend on it.

Output: a numbered session list in a proposal document called `SPRINT_PLAN.md` at the repo root. For each session:

- **Session N:** short title
- **Scope:** which files
- **Estimated units:** string count, dep count, etc.
- **Depends on:** earlier sessions (if any)
- **Sensitivity:** low / medium / high and why

### Step 4: Seed the working documents

Create or update two files:

- **`AGENTS.md`** — the project's working doc for session-to-session context. This is what Claude Code reads at the start of every session. Use the template in [`docs/example-agents-md.md`](../../docs/example-agents-md.md) from the `claude-code-multi-session-refactor` repo as a starting point. Include: project summary, stack, platform rules, conventions, code style, CI gates, commit conventions, known constraints.

- **`SPRINT_PLAN.md`** — the output of Step 3. Tracked in git so every session inherits the same roadmap.

### Step 5: Verify CI gates

Confirm that a CI workflow exists and runs on every push to sprint branches. If one doesn't, propose one. Minimum gates:

- **Static analysis** (linter / analyzer) at `0 issues` or a frozen baseline
- **Unit tests**
- **At least one build target** (web, mobile, binary — whatever the project produces)

The static-analysis gate is especially important: sprints tend to create new warnings. Treating them as a hard gate catches regressions early.

If local CI via `act` is feasible (GitHub Actions workflow + Docker available), set it up so the user can pre-validate before pushing. See the [`lessons-learned.md`](../../docs/lessons-learned.md) note about guarding any step that writes to files the user might have modified locally (e.g., `.env` credentials).

### Step 6: Commit and report

Commit `SPRINT_PLAN.md`, `AGENTS.md`, and any CI workflow changes in one go. Use a descriptive message:

```
chore: seed sprint plan, AGENTS.md, CI gates for <sprint goal>
```

Report to the user:

- The proposed session count and short titles
- The CI gate status (which gates are green at baseline, which have known-failing items to address first)
- Any hard constraints or exclusions the user should confirm before Session 1 starts
- A suggested next step: "Review `SPRINT_PLAN.md`, then run `/sprint-session-plan` to start Session 1."

## What NOT to do in kickoff

- Do not modify any source files. Investigation is read-only.
- Do not commit changes to directories outside the sprint scope.
- Do not translate, rename, or refactor anything yet. That's Sessions 1+.
- Do not skip the `AGENTS.md` setup — it's the load-bearing context for every future session.

## Handoff

When kickoff is complete, the user runs `/sprint-session-plan` to start Session 1.
