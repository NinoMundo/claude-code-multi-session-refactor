---
description: Kickoff for a localization (ARB/i18n) sprint. Extends /sprint-kickoff with l10n-specific setup.
---

You are starting a localization sprint. Run everything from `/sprint-kickoff` first, then add the l10n-specific setup below.

This command is an extension layer. Read [`commands/sprint-kickoff.md`](../sprint-kickoff.md) for the general kickoff flow. The items below are additions, not replacements.

## Additional inputs

1. **Source language.** Usually English. The ARB file will have this as the authoritative source.
2. **Target languages.** The languages you'll eventually translate to. Initially all can be English placeholders — translation is a separate pass at the end.
3. **Localization framework.** `flutter_gen_l10n` (Flutter/Dart), `react-intl` (React), `i18next`, `gettext`, etc. The commands below assume flutter_gen_l10n; adapt for other frameworks.

## Additional setup steps

### 1. Scaffold the l10n infrastructure (Session 0)

If the project doesn't have l10n set up yet, the very first session is infrastructure — before any string migration. This session:

- Adds the l10n package to dependencies.
- Creates the ARB files (`intl_en.arb`, `intl_es.arb`, etc.) with a single smoke-test key.
- Configures code generation (`l10n.yaml`, `generate: true`, output directory).
- Wires the smoke-test key into one widget to prove the pipeline works.
- Commits the generated `app_localizations*.dart` files (decision: tracked in git rather than built on every CI run).

Do not start string migration until this is green.

### 2. Establish conventions in AGENTS.md

Add an "l10n Rules" section to `AGENTS.md`. At minimum, document:

- **Naming convention**: `commonX` for strings used in 2+ domains, `domainX` for domain-specific strings. See [`docs/l10n-conventions.md`](../../docs/l10n-conventions.md) in the `claude-code-multi-session-refactor` repo.
- **Uppercase handling**: store natural case in ARB, apply `.toUpperCase()` in the widget if needed. Do not store `"SAVE"` in ARB.
- **Brand names**: every ARB key whose value contains a brand name gets an `@description` note: "X is a Y brand name — do not translate."
- **Import path**: relative (`../../l10n/app_localizations.dart`) or package-absolute — pick one and enforce.
- **Async l10n capture**: `final l10n = AppLocalizations.of(context);` before any `await`, not inside a method called after the async boundary.
- **ICU plurals**: use for any countable noun. `{count, plural, one{...} other{...}}` — not `{count} items` which ungrammatically renders "1 items."

### 3. Identify humanize helpers

Search for functions that map raw enum values (often from a database) to human-readable labels. Pattern:

```dart
String _mailStatusLabel(String status) {
  switch (status) {
    case 'scanned': return 'Scanned';
    ...
  }
}
```

These need a specific refactor: change signature to `(RawType raw, AppLocalizations l10n)` (or `(String raw, AppLocalizations l10n)`) and return l10n keys. If the helper is used in 2+ files, extract it to `lib/shared/`.

List all such helpers in the SPRINT_PLAN.md. They're session-defining: a session that touches a humanize helper should also migrate all its call sites.

### 4. Data-key strings — deferral list

Some strings are stored in the database as map keys (e.g., `completed_documents.category = 'Cedula'`, `vault_fields['Marangatu Password']`). Migrating these labels would orphan existing data. Flag them up front. They stay hardcoded until a separate session creates a code→label mapping layer.

Add a "Deferred l10n items" list to AGENTS.md's Known Constraints. The SHP-App sprint had three: `_kCompletedDocCategories`, `vaultUploadRequirements`, `VaultPerson.fields`.

### 5. Session sizing for l10n sprints

From the SHP-App experience, actual string counts were typically 2–3× the initial estimate per file. Plan for this. If `SPRINT_PLAN.md` estimates a file at 30 strings, expect 60–90 in practice. This is fine; it just means sessions run longer or split more.

Payment-sensitive, schema-sensitive, and mail-rules-sensitive files benefit from splitting into 2–3 sub-sessions even when they're not huge. Session 19–20 (Mail Detail split across UI chrome and status helper) is a good template.

### 6. Hub domains go last

The "home page" or "dashboard" domain is almost always the hub that references every other domain. Save it for the final session. By then, the shared pool is mature and the hub migration cashes in heavily.

## Handoff

When this kickoff is complete, run `/l10n-session-plan` for Session 1. The l10n-specific plan command adds reuse-search and brand-flagging on top of `/sprint-session-plan`.
