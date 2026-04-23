---
description: Session planning for an l10n sprint. Extends /sprint-session-plan with reuse search, brand flagging, ICU plural detection, humanize helper audit.
---

You are starting a session in an active l10n sprint. Run everything from `/sprint-session-plan` first, then add the l10n-specific investigation below.

This command is an extension layer. Read [`commands/sprint-session-plan.md`](../sprint-session-plan.md) for the general plan flow. The items below are additions.

## Additional investigation subagents

### Subagent A+: Enumerate strings with classification

Beyond just listing strings, classify each:

- **UI chrome** — Text(), AppBar titles, button labels, labelText, hintText, validator returns, snackbar content, dialog titles.
- **Status/enum display** — strings that map from a raw DB or API value. These need a humanize helper, not a direct l10n key. See Subagent D+ below.
- **Data keys** — strings stored as database map keys. DO NOT migrate. Flag for the deferral list.
- **Brand names / proper nouns** — strings containing "Stripe", "PayPal", "Blink", "Paraguay", service brand names, etc. Migrate but flag for `@description` non-translate note.
- **Mock/fixture content** — hardcoded demo data that isn't real UI. Skip.

### Subagent B+: Shared pool reuse search

For each candidate string, search the existing ARB for a match. Report the reuse count per session and cumulative target.

Guaranteed reuses to verify at every session (if applicable to your ARB):

- `commonCancelButton`, `commonSaveButton`, `commonAddButton`, `commonViewButton`, `commonEditButton`
- `commonEmailLabel`, `commonFullNameLabel`, `commonAddressLine1Label`, etc.
- `commonSavingLabel`, `commonLoadingLabel`
- `commonGenericError` — a generic "Error: {error}" with {error} as String placeholder
- Status keys from prior humanize-helper sessions (`commonMailStatus*`, etc.)

**commonX promotion rule**: a string qualifies for `commonX` naming only with **grep-verified evidence in 2 or more domain files**. Single-domain usage → domain-prefixed name. This rule prevents the shared pool from inflating with speculative keys.

If a string appears in this session's file AND in an existing domain file, that's grounds for in-session promotion: rename `profileX` → `commonX` in Commit B1, update the old call site, then use the renamed key in the new file's migration in Commit B2. Same-session cleanup; no deferred technical debt.

### Subagent C+: Brand-name flagging

For every string containing a brand, proper noun, or Paraguay/country-specific term, record the `@description` note. Examples:

- "Stripe" → `"Stripe is a payment processor brand name — do not translate."`
- "PayPal" → same pattern
- "Cédula" → `"Cédula is the Paraguayan national identity card — do not translate."`
- "Factura" → `"Factura is the Paraguayan term for a tax invoice — do not translate."`
- "Marangatu" → `"Marangatu is the Paraguayan tax authority's online portal — do not translate."`

### Subagent D+: Humanize helper audit

Search the session's in-scope files for any function that does enum-string → display-string mapping:

```dart
String _someLabel(String raw) {
  switch (raw) {
    case 'foo': return 'Foo Label';
    case 'bar': return 'Bar Label';
  }
}
```

For each helper found:

- Report its signature and implementation.
- Enumerate raw enum values it handles, and display strings it returns.
- List all call sites in the session's files.
- Propose a refactor: change signature to `(RawType raw, AppLocalizations l10n)` and return `l10n.keyName` for each case.
- If the helper is used in 2+ files, recommend extracting to `lib/shared/`.
- If the helper has dead cases (enum values the database never actually writes), flag for deletion. Verify by searching migrations, Edge Functions, and client code before declaring "dead."

### Subagent E+: ICU plural detection

Scan the session's files for string interpolation patterns that suggest plurals:

- `'$count items'` (always uses "items" — ungrammatical when count == 1)
- `'$count item${count == 1 ? "" : "s"}'` (manual ternary — good intent, but ARB should use ICU)
- `count == 1 ? '1 thing' : '$count things'` (branch in code — same, ICU-ify)

Propose ICU plural keys: `"{count, plural, one{1 thing} other{{count} things}}"`. Specify the `count` placeholder as `int` (not `String`) so locale-aware number formatting works.

### Subagent F+: Prior-session exclusions specific to l10n

Beyond general prior-session exclusions, watch for:

- **Error handlers** excluded from a shared-snackbar consolidation (e.g., custom red styling, hideCurrentSnackBar patterns, payment-error passthroughs where the error text comes from the gateway).
- **Shared viewers/dialogs** where a prior session explicitly kept one domain's widget separate for architectural reasons.
- **Payment SDK boundaries**: gateway-hosted UI (Stripe SetupIntent, PayPal WebView, Blink wallet deep-link) is not our UI — only our wrapper chrome migrates.

## Plan sections specific to l10n

In `SESSION_N_PLAN.md`, add these sections:

- **Reuse opportunities** — table of reuses with file:line evidence, reuse count.
- **New shared (`commonX`) promotions** — with 2+ domain evidence for each.
- **Brand-name / @description table** — every key with non-translate notes.
- **Deferred data-key strings** — any strings left hardcoded because of DB-key constraints.
- **ICU plural keys** — the ones introduced this session.
- **Humanize helper changes** — any refactor or extraction.
- **Gateway boundary** — for payment/external-service sessions, explicit table of "our UI" vs "gateway-hosted UI."

## Handoff

Same as `/sprint-session-plan`: commit the plan, push, wait for human approval. When approved, run `/sprint-session-execute`.
