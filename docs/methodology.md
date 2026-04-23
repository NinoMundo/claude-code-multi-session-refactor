# Multi-Session Refactor Methodology

This is the long-form playbook for running large refactoring sprints with a human in the loop and Claude Code doing the execution. It distills 24 sessions of real sprint work on [Sweet Home Paraguay](https://github.com/NinoMundo/SHP-App) into reusable patterns.

## Why multi-session

Large refactors don't fit in a single pull request. A 500-file string migration in one PR is unreviewable, untestable, and one mistake lands the whole thing in revert territory.

The alternative most teams default to is "one PR per file," which has its own problems: no shared context between PRs, repeated manual work, no place to accumulate conventions, easy to lose track of prior-session decisions.

The multi-session approach keeps each PR small (one or two commits, one tight scope) while sharing context across sessions via three documents:

- **`AGENTS.md`** — project conventions, invariants, known constraints. Read first at the start of every session.
- **`SPRINT_PLAN.md`** — the session roadmap. Written during kickoff.
- **`SESSION_N_PLAN.md`** — per-session investigation output. Reviewed and approved before any execution.

Git tracks all three. New sessions inherit a growing body of knowledge.

## The loop

```
[kickoff once]
  └─ investigate codebase, propose session split, seed AGENTS.md and CI gates

[per session, in order]
  ├─ 1. plan: read AGENTS.md, investigate scope, write SESSION_N_PLAN.md, commit, push
  ├─ 2. approve: human reviews plan, asks questions or amends, approves
  ├─ 3. execute: make commit(s), gate each (dart analyze / tests / build), push
  └─ 4. merge: squash-merge to main, pull, start next session
```

### Planning phase (investigate first, execute later)

The most counterintuitive part of this methodology. Every session starts with an investigation-only pass. No source files modified. Output is a `SESSION_N_PLAN.md` committed to the branch.

Why: most bad refactors are scope errors. The coder thought a file had 30 strings and it had 90. A helper function was missed. A prior-session exclusion was forgotten. A "trivial" rename turned out to shift line numbers across a dozen sibling files. Writing the plan out before touching code surfaces all of this when it's cheap to fix.

The plan has a standard structure (see `examples/session-plan-template.md`). The sections are not optional — they catch different failure modes. In particular:

- **Section E (prior-session exclusions preserved)** — forces the coder to enumerate by name every exclusion and why it's preserved. Silent regressions become impossible.
- **Section H (gates)** — lists the CI gates up front. No ambiguity about what "done" means.
- **Section I (hard constraints)** — things the coder *must not* do. Any surprise refactor in this list becomes instantly obvious when the diff is reviewed.

### Approval phase (the human brake)

After the plan is pushed, the human reviews. This is where scope errors, missed reuses, and bad abstractions get caught before any code is written. Typical review time: 5–15 minutes per plan.

Good questions the reviewer asks:

- Did the coder verify their reuse claims with actual grep evidence?
- Are prior-session exclusions all enumerated in Section E?
- Is the commit plan reasonable — not one giant commit hiding a dozen changes?
- Does anything look speculatively promoted to a shared namespace without evidence?
- Is anything outside the session's stated scope being touched?

The review is a conversation. The reviewer can ask the coder to re-investigate, amend the plan, or answer questions before approving. The approved plan is the contract for execution.

### Execution phase (small commits, gate each)

The plan says what commits to make and in what order. The coder makes them one at a time, gating each before moving on.

The gating discipline matters. A common failure mode is to make 3 commits locally, then push all three, then realize the first one has an analyzer warning that took 10 minutes to bisect. Gate each commit, push each commit, verify green before the next commit.

Local CI (via `act` or equivalent) is a huge force multiplier here. Running the full CI workflow locally before pushing catches issues in 3–8 minutes instead of waiting for a GitHub Actions round-trip. It also burns less CI budget.

### Merge phase

Squash-merge. One commit per session on main. The PR page preserves the session's internal history if anyone needs to look later.

Linear main history is essential for a sprint with 20+ sessions. A merge-commit-based history would be unreadable.

## Core patterns

### The reuse pool (commonX discipline)

Every session discovers things that could be shared. Maybe it's an ARB key, maybe a helper function, maybe a constant. The rule is:

**An item qualifies for the shared pool only with grep-verified evidence in 2 or more domain files.**

Not "I bet this'll be useful later." Not "this looks generic." *Actual* usage in actual domain files. Otherwise it stays in the domain-specific namespace.

This rule prevents the shared pool from becoming a dumping ground. In the SHP-App sprint, it kept the `common*` namespace to ~35 keys out of 586 total, all of which had real reuse.

When a candidate shows up in a second domain, the session that spots it does the promotion in-session: rename the existing domain key to `commonX`, update the old call site, then use the new `commonX` name in the current session's migration. Same-session cleanup; no deferred technical debt.

### Prior-session exclusions are invariant

Each session inherits a growing list of "don't touch this, here's why." The coder reads AGENTS.md first, and every session plan's Section E re-enumerates the exclusions relevant to that session's scope.

This prevents regressions across sessions. Session 4 excluded a handful of error snackbars from a consolidation because they had custom styling. Session 19, 12 sessions later, still knew to preserve those exclusions when migrating the surrounding file.

The discipline feels pedantic but pays off immensely at scale.

### Helper refactors pay for themselves

Some sessions are "foundation" sessions that don't add much user-visible value but set up the pattern. In the SHP-App sprint:

- Session 13 refactored `humanizeResidency()` to take an `AppLocalizations` parameter.
- Session 14 did the same for `humanizeRuc()`.
- Session 20 introduced `humanizeMailStatus()` with a similar signature.

Each helper refactor was a small session on its own, but every subsequent session that touched residency / RUC / mail status could trivially cash in the pattern.

Resist the urge to combine a helper refactor with a large migration. Keep them as separate sessions. The small foundation session is easy to review; a mixed session is a review nightmare.

### Sensitive work goes last

Payment flows, authentication, schema-sensitive code — all defer to late in the sprint. Reasons:

- The easier sessions validate the pattern and the CI gates.
- Reviewer fatigue is lowest at the end (you've seen 15 plans by then; you can evaluate sensitive ones faster than if session 1 was sensitive).
- Shared pool is mature, so sensitive sessions get maximum reuse.

SHP-App's payment work was Sessions 21–23, 15 sessions in. The mail domain (schema-sensitive) was 19–20. Home page (the hub) was last.

### The hub goes last

Whatever file or screen is the "dashboard" — the one that references every other domain — is the final session. By then the shared pool has every label it needs. That's why the SHP-App home page, estimated at 50 strings, cashed in more than 9 reuses from prior sessions for free.

## Failure modes and mitigations

### Push-forgetting

Symptom: coder makes commits locally, terminal reports success, coder reports "session done," but origin doesn't have the commits.

Mitigation: the session-complete checklist always includes `git log origin/<branch>..HEAD`. If the output is non-empty, push before declaring done. Happened twice in the SHP-App sprint (Sessions 10 and 17); the check is now a permanent part of `/sprint-session-execute`.

### Speculative commonX

Symptom: a session promotes a key to the shared pool with rationale like "this'll probably be used elsewhere later." No evidence.

Mitigation: planning phase requires **file:line evidence** for every promotion. 2+ domain files, grep output captured. The Sessions 14 and 18 plans both flagged candidates that failed this rule and stayed domain-prefixed.

### Silent CI-side clobbering

Symptom: local CI (e.g., `act`) writes stub files that overwrite the user's real local files. Gitignored, so no diff. User doesn't notice until app breaks at runtime.

Mitigation: workflow steps that write files should guard against overwriting existing ones (`if [ ! -f .env ]; then ... fi`). Post-`/act-run` checklist includes verifying credential files haven't been clobbered. Full story in `docs/lessons-learned.md`.

### Scope drift mid-session

Symptom: coder starts executing the plan, discovers something unexpected, and "fixes it while I'm there." The session's diff grows beyond what the human reviewed.

Mitigation: hard rule in `/sprint-session-execute` — if you discover something mid-execution, stop, report it, wait for the human to decide (amend plan / defer / proceed as-is). This feels slow but keeps the approval model sound.

### Forgetting AGENTS.md

Symptom: a session breaks a convention that's in AGENTS.md because the coder skipped reading it.

Mitigation: AGENTS.md is the first thing every session command (`/sprint-session-plan`, `/sprint-session-execute`) tells the coder to read. The human reviewer should double-check that any constraint violation wasn't an oversight.

## When NOT to use this methodology

- **Small refactors** — one PR, one day, one review. Don't over-engineer.
- **Rewrites** — if you're throwing away the old code, sprint methodology doesn't apply. You're not migrating, you're building new.
- **Code-review-required organizations** — if your org mandates human code review on every commit, the `/sprint-session-execute` autonomous phase may not fit your workflow. You can still use the planning phase alone.
- **Rapidly-changing codebases** — if main advances by dozens of commits per day, per-session branches rebase constantly and the overhead eats the benefit.

For 80% of the refactoring you'd otherwise dread, it's the right tool.
