---
description: Start a new session in a multi-session refactoring sprint. Investigate, produce SESSION_N_PLAN.md, commit, push, wait for approval.
---

You are starting a new session in an active sprint. This is the **investigation phase only**. No source files should be modified in this command.

## Inputs (read from context or ask if missing)

1. **Session number.** Look at existing `SESSION_*_PLAN.md` files or the most recent merge commit to figure out N.
2. **Scope.** Read `SPRINT_PLAN.md` for the planned scope of this session. Confirm with the user before proceeding if the scope has drifted.
3. **Prior-session carryovers.** Scan previous `SESSION_*_PLAN.md` files for deferred items, forward-notes, or exclusions relevant to this session's scope.

## Pre-flight

```sh
git checkout main && git pull origin main
git status   # must be clean
git checkout -b <branch-name>   # see AGENTS.md for naming convention
```

## What to do

### Step 1: Read AGENTS.md first

Always. Every session. AGENTS.md is the source of truth for project conventions, prior-session exclusions, and invariants. Do not skip.

### Step 2: Read the relevant prior session plans

At minimum, read `SPRINT_PLAN.md` and the immediately preceding `SESSION_*_PLAN.md`. If this session touches files that earlier sessions also touched (e.g., via cross-file edits or helper refactors), read those session plans too.

What to look for:

- **Forward-notes** — earlier sessions often say "when Session N touches file X, it should do Y." Honor these.
- **Exclusions** — "don't touch this call site, it's intentionally excluded from the pattern." Respect them.
- **Deferrals** — things postponed to "a future session." Check if this session is that future session.

### Step 3: Enumerate the work

Use subagents (or equivalent parallelism) to investigate in parallel:

- **Subagent A**: enumerate every unit of work in scope (strings, dependencies, findings, etc.) with file:line references.
- **Subagent B**: check for cross-domain reuse against the existing shared pool. Every candidate promotion requires grep-verified evidence in **2 or more domain files**. Single-domain usage stays domain-prefixed.
- **Subagent C**: identify brand names, proper nouns, data keys, or other strings that should NOT be migrated or should carry translator/reviewer notes.
- **Subagent D**: check for helper functions, enums, or shared state that this session's work would cross. Propose refactors with explicit signatures.
- **Subagent E**: confirm all prior-session invariants are preserved (list them by name, check each).

Run these read-only. Zero file modifications in this step.

### Step 4: Write SESSION_N_PLAN.md

At repo root. Structure:

- **A. Scope and file inventory** — line counts, primary classes/functions.
- **B. Unit inventory** — the table of work items with line numbers, current state, and proposed disposition.
- **C. Reuse opportunities** — shared items this session can cash in, with file:line evidence for each.
- **D. Gateway/boundary notes** — if applicable (payment SDKs, external APIs, generated code boundaries).
- **E. Prior-session exclusions preserved** — list each, with the line number and the reason it's excluded.
- **F. Migration mechanics** — imports to add, helper capture sites, const removals, async-await patterns.
- **G. Commit plan** — propose 1 commit or multiple. Each commit should be independently sensible.
- **H. Gates** — which CI gates the work will be validated against.
- **I. Hard constraints** — what the coder must not do in this session.

For scope splits (work too large for one session), include Section G-split: a proposal for how to split into sub-sessions and what each covers.

### Step 5: Commit and push the plan

```sh
git add SESSION_<N>_PLAN.md
git commit -m "docs: add Session <N> <short-title> plan"
git push origin <branch-name>
```

This is commit #1 of the session. The plan is tracked in git so the approver sees it on GitHub.

### Step 6: Report and wait

Report back to the human:

- Brief summary of what was found (string count, reuse count, any surprises)
- Key decisions the plan is proposing (especially anything that deviates from SPRINT_PLAN.md's original estimate)
- The branch name and plan commit SHA
- The next command: `/sprint-session-execute` once the plan is approved

**Do not execute the plan in this command.** Wait for explicit human approval before running `/sprint-session-execute`.

## What NOT to do

- Do not modify any source files outside `SESSION_N_PLAN.md`.
- Do not speculatively promote items to the shared pool without 2+ domain evidence.
- Do not skip reading AGENTS.md — no matter how many sessions deep you are.
- Do not ignore prior-session forward-notes or exclusions.
- Do not commit without pushing, and do not claim the session is ready for approval if `git log origin/<branch>..HEAD` is non-empty.

## Handoff

When the plan commit is pushed and reported, the human reviews and either approves (→ `/sprint-session-execute`), requests changes (→ revise and re-commit), or asks clarifying questions (→ answer and wait).
