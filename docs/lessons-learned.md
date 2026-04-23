# Lessons Learned

Things that went wrong during real sprints, and the checklist items that now catch them. Add to this list whenever a new sprint surfaces a new failure mode.

## .env clobber during `act` runs

**What happened.** A Flutter project's CI workflow had a step that unconditionally wrote a stub `.env` file to bundle into the web build:

```yaml
- name: Create stub .env
  run: |
    cat > export/s_h_p_home/.env <<EOF
    SUPABASE_URL=stub
    SUPABASE_ANON_KEY=stub
    EOF
```

On GitHub Actions this is fine — the runner is fresh, `.env` doesn't exist, the stub gets created, the build works.

On `act` (local GitHub Actions runner via Docker), the workflow runs against the user's real working tree. That unconditional `cat > .env` silently overwrote the developer's real Supabase credentials. Because `.env` is gitignored, nothing in `git status` or `git diff` flagged the change. The file looked unchanged on disk (same size, same modification time during the act run). The app silently failed every network call until the next `.env` reset.

**Detection.** Only symptom was the app behaving as if Supabase was down. The stub credentials caused DNS-resolution-like errors (Supabase SDK against a nonexistent project URL). Took time to trace back to `act` as the culprit.

**Mitigation — workflow level.** Guard any step that writes to a file the user might have modified locally:

```yaml
- name: Create stub .env
  run: |
    if [ ! -f export/s_h_p_home/.env ]; then
      cat > export/s_h_p_home/.env <<EOF
      SUPABASE_URL=stub
      SUPABASE_ANON_KEY=stub
      EOF
    else
      echo ".env already exists, skipping stub creation"
    fi
```

On GitHub Actions the file never exists → stub gets written → build works. On `act` locally the file exists with real creds → stub skipped → creds preserved. Zero-cost fix, zero behavior change on GitHub.

**Mitigation — checklist level.** Post-`act-run` verification: confirm any gitignored credential file still contains real values before running the app. The `/sprint-session-execute` command includes this step.

**Generalizable takeaway.** Any workflow step that writes files — not just `.env` — can silently clobber local work when run via `act`. Examples to watch for:

- `cat > config.yaml`
- `cat > local.settings.json`
- `cp template.env .env`
- Any `sed -i` against a tracked file that's been locally modified

The general fix is `if [ ! -f ... ]` guards around write operations in CI workflows. For projects that use `act` for local pre-push validation, this is an invariant of writing CI workflows.

## Push-forgetting

**What happened.** Twice during the SHP-App sprint (Sessions 10 and 17), the coding Claude made commits locally, saw the terminal report success, and reported "session complete" to the approver — without pushing the commits to origin. The approver, cross-checking via GitHub, saw the branch's tip was the plan commit and not the migration commits. Local work existed; remote work didn't.

**Why it happens.** The commit is the part that feels like "doing the work." The push is an afterthought. When a session has 3 commits and the terminal runs `git commit` three times with success, the mental model is "done." The `git push` at the end requires a separate conscious action that's easy to skip, especially at the end of a long session.

**Detection.** The approver caught it both times by running a GitHub MCP query for the branch's commits and seeing the migration commits missing. Without that check, the sessions would have been "merged" against a branch that didn't actually have the work.

**Mitigation.** Before declaring a session complete, always run:

```sh
git log origin/<branch-name>..HEAD
```

If the output is non-empty, the branch has local-only commits. Push them.

This check is a hard requirement in `/sprint-session-execute`. It catches 100% of push-forgetting failures in exchange for one additional command per session.

## Speculative commonX promotion

**What happened.** Mid-sprint, a session plan proposed promoting a string to the shared `commonX` namespace with rationale like "this will probably be used in the cart session later." No evidence — just anticipation.

**Why it's bad.** Speculative promotions inflate the shared namespace with keys that serve no actual cross-domain purpose. Over a sprint with many sessions, the `common*` prefix becomes noise rather than signal. Future sessions face a larger pool to search, reuse decisions get harder, and the naming convention loses its meaning.

**Detection.** The approver asks: "What's your file:line grep evidence for 2+ domain usage?" If the answer is "it'll probably be in file X when I migrate X," that's speculative, not evidenced.

**Mitigation.** The `commonX` rule: promotion requires grep-verified evidence in 2 or more domain files **at the time of promotion**. If the candidate is only in one domain today, it gets a domain prefix. When a second domain's migration finds the same literal, that session does the promotion — rename the existing key, update the old call site, use the renamed key in the new migration. Same-session cleanup, no deferred technical debt.

The rule is enforced in `/sprint-session-plan` and `/l10n-session-plan`. Plans proposing speculative promotions get sent back for evidence.

## Forgetting AGENTS.md

**What happened.** Occasionally a session plan would propose something that contradicted an AGENTS.md convention. E.g., using absolute imports when the convention was relative. Not malicious — just the coder skipped reading AGENTS.md because "I know the project already."

**Why it's bad.** Conventions are the glue holding a multi-session sprint together. Every session that breaks a convention makes the codebase less coherent.

**Mitigation.** Both `/sprint-session-plan` and `/sprint-session-execute` start with "read AGENTS.md first." This is the single highest-leverage instruction in the methodology. Don't skip it, even on Session 24.

Reviewers watch for convention violations during plan review. Any plan that conflicts with AGENTS.md goes back for revision — not because it can't be revisited later, but because mid-session is not the place to renegotiate conventions.

## Scope drift mid-execution

**What happened.** During execution, the coder discovered an issue outside the plan's approved scope and "fixed it while I'm there." The session's diff grew beyond what the human had reviewed.

**Example.** Session 24b discovered `prefer_const_declarations` warnings and a missing CI Java setup step during execution. Both were real issues. Both got fixed. Neither was in the approved plan.

**Why it's nuanced.** Sometimes in-flight discoveries are obviously required (e.g., a test that was failing pre-session but the coder didn't notice until execution). Other times they're genuine scope creep (e.g., "the file's naming convention bothers me, I'll rename while I'm in here").

**Mitigation.** Hard rule in `/sprint-session-execute`: if you discover something mid-execution that wasn't in the plan, stop, report it to the human, and wait for a decision (amend the plan / defer / proceed as-is). This feels slow but preserves the approval model's integrity.

In practice, the in-flight fix-ups are usually small and necessary (like the Session 24b examples above). A 60-second "found this, OK to include?" confirmation is cheaper than a broken main after a squash-merge.

## Session sizing underrun

**What happened.** Initial SPRINT_PLAN.md estimates were consistently 2–3× low. A file estimated at 30 strings turned out to have 90.

**Why it happens.** First-pass estimates are based on eyeballing or light search. They miss timeline step literals, validator returns, snackbar content, error paths, and other strings that don't show up in a quick scan.

**Detection.** Every session plan's Subagent A exhaustively enumerates strings. By the end of the investigation, the real count is known. If it's >2× the SPRINT_PLAN estimate, consider splitting.

**Mitigation.** No need to re-estimate everything at kickoff. Just accept that early estimates are directional and that a single-session scope might split into multiple sub-sessions (21→21/22/23 for Cart, 19→19/20 for Mail Detail, 24→24a/24b for Home Page). SPRINT_PLAN.md stays as a guide; per-session plans have authority over their own scope.

## Generated files out of sync

**What happened.** Early on, a session added ARB keys but forgot to regenerate `app_localizations*.dart` (via `flutter gen-l10n`) before committing. The code referenced keys that didn't exist in the generated API. CI caught it, but several minutes were lost re-running the gen-l10n step and re-pushing.

**Mitigation.** Plan's Section F ("Migration mechanics") includes the codegen command as an explicit step. Execution runs it before the first commit that references new keys. For ARB sprints: `flutter gen-l10n` after every ARB change, before any code change that uses the new keys.

---

## Adding to this file

Run into something new during a sprint? Document it here. Structure:

- **What happened** (narrative)
- **Why it's bad** / **Why it happens**
- **Detection** (how did you notice)
- **Mitigation** (what the methodology does to catch it next time)

If the mitigation belongs in a subagent command, add it there too. This file is the source of truth for lessons learned; the commands reference back here for context.
