# L10n Conventions

Conventions for localization sprints, distilled from the SHP-App experience. These layer on top of the general methodology in `methodology.md`.

## Naming

### commonX vs domainX

- **`commonX`** — strings used in 2+ domain files, grep-verified at the time of promotion.
- **`domainX`** — strings scoped to one domain. Examples: `authEmailLabel`, `cartCheckoutButton`, `profileSaveButton`.

A string qualifies for `commonX` **only when grep actually finds it in 2+ domains**. Anticipated-future-use does not count. Single-domain usage stays `domainX` until a second domain emerges, at which point the promoting session does an in-session rename + call-site update in a dedicated commit (e.g., Commit B1 before the main migration in B2).

### Status families

For status-like values (mail status, request status, residency status, RUC status), use a family prefix:

- `commonMailStatus<Value>` — e.g., `commonMailStatusScanned`, `commonMailStatusForwarded`
- `commonResidencyStatus<Value>`
- `commonRucStatus<Value>`

The prefix keeps related keys grouped alphabetically in the ARB file, which is nice for translators.

### Case and capitalization

Store natural case in the ARB. If the UI renders a string uppercase (e.g., a `.toUpperCase()` chip), do the transform **in the widget**, not in the ARB. Rationale: uppercase rules vary by locale; the transform belongs at display time.

Bad: `"commonSaveButton": "SAVE"`
Good: `"commonSaveButton": "Save"` + `Text(l10n.commonSaveButton.toUpperCase())` at the call site.

## Brand names and proper nouns

Every ARB key whose value contains a brand name, product name, or untranslatable term gets an `@description` note telling the translator not to translate it.

```json
"accountingUploadFacturaTitle": "Upload Factura",
"@accountingUploadFacturaTitle": {
  "description": "FAB label for uploading a client factura scan. 'Factura' is the Spanish/Paraguayan term for a handwritten tax invoice — do not translate."
}
```

Common categories:

- **Payment processors**: Stripe, PayPal, Wise, Blink
- **Paraguayan terms**: Factura, Cédula, RUC, Marangatu, CIIU
- **Your own brand**: "Sweet Home Paraguay," abbreviations like "SHP"
- **Technical names**: Bitcoin, BTC Lightning, WhatsApp

Do this as you migrate, not after. Going back to add `@description` notes across 500 keys is painful; adding them one-at-a-time during each session migration is trivial.

## ICU plurals

Use ICU plural form for any countable noun. Do not use `{count} items` (ungrammatical when count == 1).

```json
"profileDocumentsUploadedCount": "{count, plural, one{1 document uploaded} other{{count} documents uploaded}}",
"@profileDocumentsUploadedCount": {
  "description": "Vault person-card subtitle showing count of documents. ICU plural handles 'one' vs 'other' correctly.",
  "placeholders": { "count": { "type": "int" } }
}
```

The `count` placeholder should be typed as `int`, not `String`, so the locale-aware number formatter fires (e.g., European locales use `.` as thousands separator).

When a plural is introduced for the first time in a project, add a note to AGENTS.md saying "use ICU for countable nouns by default from here on." This prevents later sessions from re-introducing the `{count} items` bug.

## Humanize helpers

Pattern: a function that maps raw enum/status values (usually from a database) to human-readable labels.

### Refactor signature

Original:
```dart
String _mailStatusLabel(MailItemRow mail) {
  final status = mail.status.toLowerCase();
  if (status.contains('marked_for_forward')) return 'Marked for forwarding';
  if (mail.contentPdfUrl != null) return 'Scanned';
  ...
}
```

Refactored:
```dart
String humanizeMailStatus(MailItemRow mail, AppLocalizations l10n) {
  final status = mail.status.toLowerCase();
  if (status.contains('marked_for_forward')) return l10n.commonMailStatusMarkedForForwarding;
  if (mail.contentPdfUrl != null) return l10n.commonMailStatusScanned;
  ...
}
```

Key changes:

- Name change from private (`_mailStatusLabel`) to public (`humanizeMailStatus`) if the helper will be used across files.
- Parameter added: `AppLocalizations l10n`.
- Return values replaced with `l10n.keyName` calls.
- **Ordering invariants preserved.** If the original used `contains()` for substring matching with order-sensitive precedence (marked_for_forward must be checked before forward), the refactored version preserves the same order. Switch/case is only usable if the original did exact matching.

### Extraction to lib/shared/

When the helper is used in 2+ files, extract to `lib/shared/<helper_name>.dart`. Trigger:

- **2 files**: keep in the domain file, accept the cross-file import.
- **3+ files**: extract to `lib/shared/`. Update all imports.

The SHP-App precedent: `humanizeMailStatus` stayed in `mail_detail_widget.dart` (1 consumer) → admin_user_detail imported it (2 consumers, acceptable cross-page import) → when home_page became the third consumer in Session 24, extraction to `lib/shared/humanize_mail_status.dart` happened.

### Enum-label functions without row context

Some helpers take just a `String` and don't need the full row:

```dart
String humanizeRuc(String raw, AppLocalizations l10n) { ... }
```

Others need the full row because display logic depends on multiple fields (e.g., `humanizeMailStatus` inspects `mail.contentPdfUrl` to differentiate scanned-with-PDF from scanned-without-PDF):

```dart
String humanizeMailStatus(MailItemRow mail, AppLocalizations l10n) { ... }
```

Pick the minimum signature the logic requires. Don't pass the whole row if only the status string matters.

## Data-key strings (the deferral class)

Some strings are stored in the database as map keys:

```dart
// vault_people.fields is stored as Map<String, String>
// where keys are things like 'Name', 'Cedula Number', 'Marangatu Password'
// Migrating the display label to ARB would orphan existing row data.
```

These stay hardcoded. Flag them in AGENTS.md's "Known Constraints" section. Migration requires a separate code→label mapping layer, which is a schema-sensitive task deferred to its own session.

Common examples from the SHP-App sprint:

- `_kCompletedDocCategories` — document category enum stored as DB strings.
- `vaultUploadRequirements()` — required document names as map keys.
- `vaultSubmissionTextFields()` — form field labels as map keys.
- `VaultPerson.fields` — person profile fields as map keys.

If the sprint plans to fix the data model (add a canonical code → localize the label), schedule that as its own session at the end.

## Async l10n capture

```dart
Future<void> _saveAddress() async {
  final l10n = AppLocalizations.of(context);  // capture BEFORE await
  try {
    await supabase.from('addresses').insert(...);
  } catch (e) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(l10n.addressSaveFailedError)),  // use the captured l10n
    );
  }
}
```

Capturing `l10n` after an `await` risks using a stale context if the widget has been disposed. Capturing before the await avoids this. Even with `mounted` checks, the idiomatic pattern is to capture synchronously before any async boundary.

Special cases:

- **initState-triggered async**: can't capture at method top because widget isn't fully mounted. Capture inline, after the `!mounted` check inside the method body.
- **Bottom sheet builders**: the builder callback receives its own context, which shadows outer context. Capture `l10n` from the outer context before `showModalBottomSheet` and pass it into the builder scope via closure.

## Brand-name and translator @description examples

```json
"commonCedulaOrPassportHint": "Cédula or Passport",
"@commonCedulaOrPassportHint": {
  "description": "Hint text for a form field asking for ID type. 'Cédula' is the Paraguayan national identity card — do not translate."
}

"accountingFacturaRequestedSnackbar": "Factura requested. Your accountant will upload it shortly.",
"@accountingFacturaRequestedSnackbar": {
  "description": "Ephemeral SnackBar shown after a successful requestFactura() RPC. Note period (.) at end — distinct from accountingFacturaRequestedConfirmation which uses em-dash (—). 'Factura' = do not translate."
}

"cartBlinkInvoiceExpiredMessage": "This invoice has expired — please refresh to get a new one.",
"@cartBlinkInvoiceExpiredMessage": {
  "description": "Status label when Blink Lightning invoice has expired. Blink is a Bitcoin Lightning wallet service brand — do not translate. Transactional UX — do not alter meaning."
}
```

Good `@description` notes tell the translator two things: what the key is for (context), and what must be preserved (brand names, tone, placeholder structure). Over-descriptive is fine; under-descriptive is bad.

## Generated files

`flutter_gen_l10n` regenerates `app_localizations.dart`, `app_localizations_en.dart`, etc. after every ARB change. Decision: commit these files to git.

Reasons:

- CI doesn't have to re-run codegen on every build.
- Diffs show the generated API surface changing, which is useful context during code review.
- Developers checking out the repo don't need to run codegen before the project compiles.

Alternative (regenerate on every CI build): saves a few hundred lines of churn but requires `flutter gen-l10n` as a prerequisite for every new contributor. Tradeoff isn't worth it for this use case.

## Session sizing for l10n

Rule of thumb: **actual string count is 2–3× the initial estimate** per file. Plan for this.

SHP-App sprint examples:

| File | Estimated | Actual |
|---|---|---|
| profile_tools | 35 | ~85 |
| home_page | 50 | ~130 |
| cart_tools + submission_setup | 35 | ~69 |
| mail_detail | 20 | ~50 |

Estimates live in `SPRINT_PLAN.md` and are used only for ordering (not committing to a specific per-session count). When a session's actual count blows past its estimate, the session plan splits — Session 21 split into 21/22/23 for Cart, Session 19 split into 19/20 for Mail Detail, Session 24 split into 24a/24b for Home Page.

## Scope edges

### Gateway-hosted UI

For payment flows: the gateway (Stripe, PayPal, Blink, Wise) renders some UI itself inside WebViews or native sheets. Those strings are not ours — don't migrate them. Only our wrapper UI migrates.

Examples from SHP-App:

- **Stripe SetupIntent native sheet** — gateway-hosted, we don't touch.
- **PayPal WebView approval flow** — gateway-hosted, we don't touch.
- **Blink wallet deep-link destination** — external app, we don't touch.
- **Stripe/PayPal/Blink brand-name tiles in the payment selector** — our UI, we migrate.

The session plan should include a gateway boundary table for any payment-touching session.

### Mock/fixture data

Chat pages often have hardcoded demo messages for UI development. Exclude these from migration — they'll be replaced when real data arrives, and migrating them creates orphaned keys.

SHP-App's `home_page_widget.dart` had about 20 lines of mock community chat messages that were cleanly excluded per the Session 10 pattern.
