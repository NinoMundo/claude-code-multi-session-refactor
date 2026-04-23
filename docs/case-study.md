# Case Study: SHP-App Localization Sprint

24 sessions. 586 ARB keys. 13 domains. Zero hardcoded user-facing strings remaining.

This is the story of the sprint that developed the methodology in this repo. It ran April 21–22, 2026, on [Sweet Home Paraguay](https://github.com/NinoMundo/SHP-App) — a Flutter + Supabase app for expats establishing Paraguay residency. Single operator (Nino), production users, real business depending on the app not breaking mid-sprint.

## The starting point

The app was entirely English. ~500 user-facing strings scattered across 13 domain files. No l10n infrastructure. The goal: every string in an ARB file, Spanish translation to be a separate pass afterward.

Pre-sprint state:

- No `AGENTS.md` yet — it was written during Session 0's kickoff.
- No CI gates on `dart analyze` — analyzer had 76 warnings, with `continue-on-error: true` hiding them.
- A 10-finding audit document (`AUDIT.md`) pointing at various code quality issues. Sessions 1–7 addressed the audit before l10n started.

## Sessions 1–7: the audit pass

Before any localization, the audit findings. Seven sessions, each small, each merged before the next started:

1. Async resilience — 3 bang-operators guarded, 2 silent catches surface errors.
2. Dependency cleanup — 20 unused deps removed.
3. Shared `DocumentViewerPage` — 5 viewer classes consolidated. Mail domain excluded (first prior-session exclusion noted).
4. Shared `showErrorSnackBar` helper — 39 call sites migrated. 6 mail-domain sites excluded with custom styling.
5. Humanize helpers made public — `humanizeRuc`, `humanizeResidency`.
6. Raw status string audit — one site migrated, one deferred (the Session 6 mail-admin badge, which came back to life in Session 20 fourteen sessions later).
7. Analyzer burn-down — 76 → 0. `continue-on-error: true` removed. `dart analyze lib/` became a hard CI gate.

By Session 7, the project had an `AGENTS.md`, a working CI, and a clean analyzer baseline. Only then did localization start.

## Session 8: infrastructure

Before any strings migrated: scaffold the l10n system. `flutter_gen_l10n` configured, empty `intl_en.arb` and `intl_es.arb` created, a single smoke-test key wired into the home screen. Decision: commit generated `app_localizations*.dart` files to git (faster CI, easier onboarding).

This session added zero user-value. It was foundation. But nothing past this point would have worked without it.

## Sessions 9–12: simple domains first

Auth, Chat, Product Detail, Request Detail. Small files. Safe patterns. By Session 12, the pattern was proven:

- Read AGENTS.md first.
- Write `SESSION_N_PLAN.md` read-only.
- Plan identifies reuses with grep evidence.
- Commit plan, push, approve.
- Execute commits, gate each, push each.
- Squash-merge.

Sessions 9–12 cumulative: ~50 keys, 0 shared promotions (nothing to share yet).

## Sessions 13–14: foundation helpers

The humanize refactors. `humanizeResidency()` and `humanizeRuc()` changed from `String → String` to `(String, AppLocalizations) → String`, returning l10n keys. Each was a small session — ~50 keys, but the investment paid off in every subsequent session that touched residency or RUC.

Session 14 also introduced the first `commonX` promotions: 10 keys moved from domain-specific namespaces (`residencyX`, `rucX`) to shared. Grep-verified at 2+ domains each. Residency's old keys deleted in-session to avoid deferred technical debt.

## Session 15: the payoff session

Admin Panel. 91 net new keys, largest single session of the sprint. 8 new `commonX` promotions specifically naming future sessions ("Accounting will reuse this," "Profile will reuse this"). The coder didn't just promote — they grep-verified each promotion against specific line numbers in future target files. That prediction turned out to be load-bearing: Session 16 (Accounting) reused all 7 cited commonX keys at the exact lines Session 15 had called out.

The `commonX` discipline from Session 14 had paid off. Every subsequent session had a growing pool to draw from.

## Sessions 17–18: the high-reuse era

Profile (58 new keys). Service Active (47 new keys). By Session 18 the shared pool had ~35 `commonX` keys, and reuse counts were climbing. Session 18 hit 19 reuses in one session — triple what Session 13 had managed.

Session 17 also did something useful: it found 140 lines of dead code (`_AddCardDialog`, orphaned after a Stripe SetupIntent migration replaced it) and deleted it. The migration-plus-cleanup pattern became a recurring theme: if a session discovers dead code in its scope, it deletes it in a dedicated commit.

## Sessions 19–20: sensitive domain splits

Mail Detail split across two sessions: UI chrome (19), then the `humanizeMailStatus` helper (20). The split was intentional — packing a helper refactor and a 40-string migration into one session would have been too much for a single review.

Session 20 closed a 14-session-old deferral: the admin mail-badge at `admin_user_detail_widget.dart:332`, deferred back in Session 6, finally got its `humanizeMailStatus(item, l10n)` resolution. The deferral survived the intervening 14 sessions because every session plan's Section E re-enumerated it.

Session 20 also investigated and confirmed a piece of dead code in the home page's `_statusLabel` — an `'opened'` case that no DB migration or Edge Function ever wrote. Deferred to Session 24 for deletion.

## Sessions 21–23: payment-sensitive work

Cart, split three ways: post-payment setup (21), payment method selector + PayPal (22), Blink Lightning live invoice (23). Complexity-first ordering — simpler sessions validated the pattern before the live transactional work.

Session 21 reused 17 keys from the accumulated pool, validating the commonX investment at scale. Session 22 introduced a UX improvement mid-sprint: the generic "Payment capture failed: {error}" was replaced with "The transaction failed. Try again or use a different form of payment." The technical error went to `debugPrint`; users got actionable copy.

Session 23 similarly softened the "Invoice expired" label into "This invoice has expired — please refresh to get a new one."

## Session 24: the hub

Home page. Largest file, most references, saved for last.

The session split into 24a (foundation — extract `humanizeMailStatus` to `lib/shared/`, delete dead code) and 24b (77 new ARB keys, full migration). The 24a→24b split enabled clean git history and reduced review surface area.

Session 24b caught a class of issues that shipped silently: `prefer_const_declarations` warnings from Flutter's analyzer on the new ARB-wired widgets. The initial push broke CI. Fix-up commit, re-push, green.

Session 24 also cashed in **9 reuses from the `commonMailStatus*` pool alone**, plus 5 more from other domains. The hub migration was the easiest session of the sprint in terms of key creation, because by then almost everything it needed existed.

## The final numbers

- **24 sessions** (Sessions 1–7 audit + Session 8 infra + Sessions 9–24 localization)
- **586 ARB keys** (~35 shared `commonX`, ~551 domain-specific)
- **13 domains fully migrated**: Auth, Chat, Product Detail, Request Detail, Service Residency, Service RUC, Admin Panel, Accounting, Profile, Service Active, Mail Detail, Cart, Home Page
- **3 humanize helpers** refactored to l10n-aware: `humanizeRuc`, `humanizeResidency`, `humanizeMailStatus`
- **5 ICU plural keys** (first introduced in Session 17)
- **140+ lines of dead code removed** (`_AddCardDialog`, `_cardBrandForNumber`, `_MailboxAddressCard`, `'opened'` mail status case)
- **0 hardcoded user-facing strings** remaining in app code

## What worked

### Plan-first, execute-second

Every session plan caught something. Wrong scope estimates, missed helper refactors, unconsidered prior-session exclusions, speculative `commonX` promotions that failed the 2-domain evidence test. The average plan-to-approval cycle was 10 minutes of human review; each one saved several times that in downstream confusion.

### CommonX discipline paid off exponentially

The `commonX` pool grew from 0 to ~35 over 16 sessions, and reuse counts per session climbed from 0 to 19. Without the evidence-required promotion rule, the pool would have ballooned with speculative keys; with it, every shared key earned its place.

### Prior-session exclusions held for 20+ sessions

The Session 3 `_PdfViewerScreen` boundary, the Session 4 showErrorSnackBar exclusions for mail_detail — these survived every subsequent session without being accidentally regressed, because every session plan's Section E re-enumerated them.

### Sensitive work last

Sessions 21–23 (payment) and 24 (hub) benefited from everything the earlier sessions had built. The pattern was proven, the shared pool was mature, the CI was hard-gated. Payment work didn't become the experiment where those things got figured out.

### Small copy improvements along the way

PayPal failure message. Invoice expired message. Small changes, opportunistically batched with l10n migration. Users benefit without requiring dedicated UX sessions.

## What could have gone better

### Push-forgetting

Twice (Sessions 10 and 17) the coder reported "session complete" with local commits that hadn't been pushed to origin. Caught both times by the approver cross-checking via GitHub MCP. The post-hoc fix: `git log origin/<branch>..HEAD` became a required check in `/sprint-session-execute`.

### Silent `.env` clobber

Late in the sprint, a local `act` run overwrote the project's real `.env` credentials with the CI workflow's stub values. Gitignored file, no diff, no warning — detected only when the app silently failed all network calls post-run. The fix lives in `docs/lessons-learned.md` and `/sprint-session-execute` now has a post-`/act-run` credential-verify step.

### Estimate inflation

The initial SPRINT_PLAN.md estimates were consistently 2–3× low. `profile_tools` estimated at 35 strings, actual 85. `home_page` estimated at 50, actual ~130. This isn't a methodology flaw, just a calibration note: for your first sprint, expect estimates to be low and plan for splits.

### One-session scope creep

Session 24b was approved for 77 keys + _MailboxAddressCard cleanup. During execution, it also needed a `prefer_const_declarations` fix-up and a CI Java-setup addition that weren't in the plan. These were legitimate requirements surfacing at execution time, but they landed in the squash-merge without explicit pre-execution approval. A stricter reading of the methodology would have paused, amended the plan, and re-approved. A practical reading says the fix-ups were small and necessary. Judgment call.

## Artifacts from the sprint

- The 24 session plans, each tracked in git on their respective branches (preserved in PR page git history after squash-merge).
- `AGENTS.md` — grew from zero to a ~500-line working document over the sprint.
- `AUDIT.md` — the original 10-finding audit, Sessions 1–7 addressed.
- `SPRINT_PLAN.md` — the session roadmap, continuously updated.
- `lib/shared/humanize_mail_status.dart` — first extraction to the shared directory.
- `lib/l10n/intl_en.arb` + `intl_es.arb` — 586 keys.

All preserved in the public SHP-App repo.

## Conclusions

For a refactor this size, multi-session with a planning phase was the only realistic approach. A single giant PR would have been unreviewable; one PR per file without shared context would have drowned in duplicate work. The methodology's overhead (plan → approve → execute → merge per session) felt heavy at Session 1 and felt weightless by Session 20 — the approver gets faster, the coder gets more pattern-aware, the shared pool handles more cases automatically.

Total wall-clock: about 24 hours across two days of focused work. Roughly equal time between planning-and-approval (me) and execution (Claude Code). Zero reverts. Zero production incidents during the sprint window. One `.env` incident that was found and fixed within minutes.

This methodology is what this repo exists to share.
