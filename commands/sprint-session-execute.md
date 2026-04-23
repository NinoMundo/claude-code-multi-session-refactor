---
description: Execute an approved SESSION_N_PLAN.md. Small commits, gate each, use /act-run pre-push.
---

The plan has been approved. Execute it. One commit at a time, gating each before moving on.

## Pre-flight

1. **Confirm you're on the session branch** and it's up to date with origin.
2. **Re-read `SESSION_N_PLAN.md`** from disk — don't trust memory. Plans sometimes get amended during approval discussion; the commit in git is the source of truth.
3. **Re-read `AGENTS.md`** if it's been a while (e.g., you're returning from a break mid-session).

## Execution loop

For each commit the plan proposes (section G of the plan):

### Step 1: Make the commit locally

- Apply the plan's changes for this commit only.
- Use the exact import convention, key naming, helper signatures the plan specifies.
- Preserve all prior-session exclusions listed in Section E.
- Remove `const` where required by the migration (localized strings are not const).
- Run the code generator if the plan requires it (e.g., `flutter gen-l10n` after ARB changes).

### Step 2: Run gates locally

The gates listed in the plan's Section H. At minimum:

- Static analysis (linter/analyzer) must be at baseline (typically 0).
- Tests must pass.
- At least one build target must build.

If `/act-run` is available in this Claude Code environment (local `act` + Docker setup), prefer it. It runs the full GitHub Actions workflow locally and catches any issue the actual CI would.

**After any `/act-run`, verify files the CI creates as stubs haven't overwritten your real local values.** In particular: if the CI workflow writes `.env` or similar credential files as part of the build, local `act` runs will write the same stubs over your real credentials. This is silent (gitignored files don't appear in diffs) and will cause the app to fail at runtime with authentication errors. Check the file contents or reset from your password manager / `.env.example` after each `/act-run`.

See [`docs/lessons-learned.md`](../../docs/lessons-learned.md#env-clobber-during-act-runs) in the `claude-code-multi-session-refactor` repo for the full story.

### Step 3: Push and wait for remote CI

```sh
git push origin <branch-name>
```

Then wait for GitHub Actions to report green on this commit. Don't move to the next commit until CI is green — if something broke, it's cheaper to fix the one commit than to debug a multi-commit branch.

### Step 4: Report the commit

Report the commit SHA, the gate results (both local and remote), and any deviations from the plan.

Repeat for each commit in the session.

## Push-forgetting check

Before declaring the session complete, always run:

```sh
git log origin/<branch-name>..HEAD
```

If the output is non-empty, there are local commits that haven't been pushed. Push them before reporting "done."

This catches a common failure mode: making a commit locally, seeing the terminal succeed, and mentally moving on without the `git push` step. It's happened twice in the SHP-App sprint (Sessions 10 and 17). A one-line check at the end of every session prevents it.

## Handoff

When all commits in the plan are pushed and green on GitHub Actions, the session is ready to merge. The human runs `/sprint-merge` to open the PR and squash-merge.

## Hard constraints during execution

- Do not add work that wasn't in the approved plan. If you discover something mid-execution, stop, report it, and wait for the human to decide: amend the plan, defer to a future session, or proceed as-is.
- Do not skip gates. If static analysis goes from 0 to 5, fix it before pushing — don't just note it and move on.
- Do not re-architect. Surprise refactors outside the plan's scope break the trust model.
- Do not merge your own PR. The human reviews via GitHub UI and runs `/sprint-merge` when ready.
