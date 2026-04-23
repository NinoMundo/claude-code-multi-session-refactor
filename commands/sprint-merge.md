---
description: Open PR, squash-merge, delete branch, pull main, report new main SHA.
---

The session's commits are pushed and green on CI. Close out the session.

## Pre-flight

Confirm:

- All commits for this session are on origin (`git log origin/<branch>..HEAD` is empty)
- GitHub Actions is green on the tip commit
- The branch is up to date with main (rebase if needed, or let the merge handle it)

## Steps

### 1. Open the PR

```sh
gh pr create --base main --head <branch-name> \
  --title "<descriptive title matching the session's work>" \
  --body "$(cat SESSION_<N>_PLAN.md | head -50)"
```

Or use a custom body that summarizes the session: scope, commits, deviations from plan, gate results.

### 2. Squash-merge

```sh
gh pr merge <pr-number> --squash --delete-branch
```

Why squash: the plan-commit plus execution-commits compress into one commit on main. The branch history is preserved in the PR page. Main stays linear and readable.

### 3. Update local main

```sh
git checkout main
git pull origin main
```

### 4. Report the new main SHA

Report to the human:

- PR URL
- Squash-merge SHA (the new main tip)
- Total ARB / dep / finding count after this session (if applicable)
- Reuse count for this session
- Any deferred items that carried forward into a later session
- Suggested next command: `/sprint-session-plan` for Session N+1

## What NOT to do

- Do not merge via a non-squash strategy. Merge commits and rebase merges muddy the main history for a sprint like this.
- Do not delete the branch before the merge completes — `gh pr merge --delete-branch` handles this as one operation.
- Do not skip the `git pull` after merge. Future sessions start from main; if local main is stale, they'll branch off the wrong commit.
