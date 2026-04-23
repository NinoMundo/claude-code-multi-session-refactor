# Example AGENTS.md Template

This is a template for an `AGENTS.md` file at the root of a project that uses the multi-session refactor methodology. It's the first thing Claude Code reads at the start of every session. Keep it current.

Copy the structure and fill in your project's specifics. Remove sections that don't apply.

---

# AGENTS.md

## Project summary

<One paragraph: what this project is, who it serves, what stage it's in.>

Example: "Sweet Home Paraguay (SHP) is a service business app helping expats establish residency in Paraguay. Built in Flutter with a Supabase backend, iOS + Android + Web. Single-operator product; production users are paying customers."

## Stack

- Language + framework: <Flutter 3.X / React 18.X / etc.>
- Backend: <Supabase / Firebase / custom / etc.>
- Platforms: <iOS, Android, Web — which are live, which are beta>
- Router: <go_router 12.X / react-router / etc.>
- State: <Provider / Riverpod / Redux / etc.>
- Localization: <flutter_gen_l10n / react-intl / etc.> (if applicable)

## Platform rules

<Per-platform constraints. Example: "iOS builds via Codemagic cloud CI (no local Mac). Android builds via GitHub Actions. Web via Docker/Nginx on VPS.">

## Domain rules (critical invariants)

List the business rules that MUST be preserved through any refactor.

Example (from SHP-App's Mail Rules):

- Two-phase unread logic: mail_items unread state is computed across two separate state reads (envelope + contents). Any refactor preserves both reads.
- State transitions: `received` → `scan_requested` → `scanned` is the canonical lifecycle. Do not introduce new terminal states without schema changes.
- Admin badge reflects client-side status verbatim via `humanizeMailStatus(item, l10n)` — no separate admin-side labels.

Do this for every domain that has non-trivial business logic. These become the prior-session-exclusions baseline.

## Styling rules

- Font family: <Montserrat / etc.>
- Primary color: <#HEXCODE>
- AppBar color: <#HEXCODE>
- Button variants: <which widget, which style>
- Do not use: <any anti-patterns specific to your project>

## Code style

- Import convention: <relative / package-absolute — pick one>
- Max line length: <80 / 100 / 120>
- Linter config: <analysis_options.yaml / eslintrc.json>
- Formatter: <`dart format` / `prettier` — enforced on CI or pre-commit>

## Testing

- Test framework: <flutter_test / jest / etc.>
- Minimum coverage: <if any>
- Manual test areas: <things that aren't covered by automated tests and require hand-verification before release>

## CI gates (MUST pass for any PR to merge)

- Static analysis: <`dart analyze lib/` at 0 issues / `eslint` at 0 errors>
- Unit tests: <pass>
- At least one build target: <web / iOS / Android / binary>
- <Any other gates>

## Commit conventions

- Conventional Commits: `refactor(l10n):`, `fix(auth):`, `feat(cart):`, `chore:`, `docs:`
- Commit title ≤ 72 chars
- Body explains WHY, not just WHAT
- Claude Code commits signed with `Co-Authored-By: Claude <noreply@anthropic.com>`

## Sprint working documents

- **`SPRINT_PLAN.md`** — the current sprint's session roadmap.
- **`SESSION_<N>_PLAN.md`** — per-session investigation output, committed at the start of every session before execution.

At the start of every session, Claude Code reads:

1. This file (`AGENTS.md`)
2. `SPRINT_PLAN.md`
3. The immediately preceding `SESSION_<N-1>_PLAN.md` (and any earlier plans whose files overlap with this session's scope)

## Known constraints

Long-running blockers, deferred items, and "don't fix this here" notes. Add to this list as sessions discover things.

Examples:

- **`<feature>` is blocked** pending `<external dependency>`. UI visible but non-functional.
- **`<data>` is stored as DB keys**, so display labels can't be localized without a code→label mapping layer (deferred to its own session).
- **`<third-party SDK>` web path is gapped** vs mobile. Product-blocked from being fixed in a sprint context.

## Deferred sessions / technical debt

Things the current sprint is explicitly NOT addressing. Listed so future sprints can pick them up.

Examples:

- Spanish translation pass (all 586 keys have English placeholders in `intl_es.arb`; translator fills in)
- Data-key label separation for `VaultPerson.fields` etc.
- `logger` migration (Session 7 follow-up)
- Node.js 20 → 24 for GitHub Actions (June 2026 deadline)

## Prior-session exclusions (critical — read every time)

A growing list of "don't touch this; here's why." Each session that adds an exclusion updates this section. Each subsequent session reads it.

Examples (from SHP-App):

- **Session 3 exclusion**: `mail_detail_widget.dart`'s `_PdfViewerScreen` is NOT swapped for the shared `DocumentViewerPage`. Reason: mail domain boundary. Strings inside `_PdfViewerScreen` ARE migrated to ARB; only the widget architecture is preserved.
- **Session 4 exclusion**: six error snackbars in `mail_detail_widget.dart` use raw `ScaffoldMessenger.of(context)` rather than the shared `showErrorSnackBar` helper. Reason: custom styling / `hideCurrentSnackBar` patterns. Content strings migrated to ARB; wrappers preserved.
- **Session 4 exclusion**: `cart_tools.dart:469` `response.errorMessage!` is a dynamic passthrough from the payment gateway. Nothing to migrate; no ARB key applies.
- **Session 6 deferral**: admin-side mail status badge at `admin_user_detail_widget.dart:332` waited until `humanizeMailStatus` existed (Session 20).

---

## Tips for keeping this file healthy

- Keep it under 1000 lines. If it's growing unboundedly, some of the content belongs in a separate doc.
- Update it at the end of every session that changes a convention or adds an exclusion.
- Review it quarterly. Prune items that no longer apply.
- Link to other docs (`CONTRIBUTING.md`, `SECURITY.md`, etc.) rather than duplicating their content.
