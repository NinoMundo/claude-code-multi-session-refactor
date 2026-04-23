# Session N — <Short Title>

**Branch:** `<branch-name>`
**Status:** Investigation complete — awaiting approval before STEP 2 (execution)
**Prerequisite:** Session <N-1> merged (main SHA `<sha>`)

---

## A. Scope and File Inventory

<Which files are in scope this session? Line counts, primary classes, relevant widgets/functions.>

| File | Lines | Primary Classes/Functions |
|---|---|---|
| `path/to/file.dart` | ??? | `ClassName`, `functionName()` |

---

## B. Unit Inventory

<The table of work items. One row per string / dep / finding / etc.>

| # | Line | Current literal | Proposed key / action | Reuse? |
|---|---|---|---|---|
| 1 | ??? | `'Example string'` | `domainSomethingKey` | — |
| 2 | ??? | `'Another string'` | (reuse) `commonCancelButton` | ✓ |

**Total units:** ???
**New keys/items:** ???
**Reuses:** ???

---

## C. Reuse Opportunities (l10n: commonX pool cash-in)

<If this is an l10n session, list every reuse with file:line evidence. Reuse count is the session's headline metric.>

| Key | EN value | Used at |
|---|---|---|
| `commonViewButton` | "View" | file.dart:??? |
| `commonCancelButton` | "Cancel" | file.dart:??? |

**Total reuses this session:** ???
**Target:** <2, 5, 10, 15, etc. based on session scope>

---

## D. Gateway / Boundary Notes

<For payment, auth, external-SDK, or schema-sensitive sessions: which strings are "ours" vs "not ours." Which strings migrate vs preserve.>

| String | File:Line | Rendered by | Migrate? |
|---|---|---|---|
| `'Example'` | file.dart:??? | Our UI | ✓ Yes |
| `'Example from gateway'` | file.dart:??? | Stripe/PayPal/etc. hosted UI | ✗ No |

---

## E. Prior-Session Exclusions Preserved

<Enumerate every exclusion from earlier sessions that applies to this session's scope. Section E is the regression-prevention firewall.>

- **Session 3 exclusion**: `mail_detail_widget.dart`'s `_PdfViewerScreen` architecture unchanged. Strings inside it are migrated to ARB; widget itself not swapped for shared `DocumentViewerPage`.
- **Session 4 exclusion**: 6 raw `ScaffoldMessenger` wrappers in `mail_detail_widget.dart` preserved. Only content strings migrated.
- **Session 6 deferral**: `admin_user_detail_widget.dart:332` admin mail badge handled by this session via `humanizeMailStatus`.
- <Other exclusions relevant to this session...>

---

## F. Migration Mechanics

<Imports, l10n capture points, const removals, async-await patterns, codegen steps.>

### Import

```dart
import '../../l10n/app_localizations.dart';
```

### l10n capture per method

| Method | Pattern |
|---|---|
| `_SomeWidget.build(BuildContext context)` | `final l10n = AppLocalizations.of(context);` |
| `_asyncMethod()` | Capture before first `await` |

### Codegen

```sh
flutter gen-l10n
```

### const removals

- `Text(l10n.something)` is not `const` — remove `const` from wrapping `Text`/`PopupMenuItem`/`InputDecoration`.

---

## G. Commit Plan

<1 commit or multiple commits. Each commit should be independently sensible. For in-session cleanup (rename existing key → then use renamed key), separate commits give clean git history.>

### Commit 1 (this session's STEP 1):

```
docs: add Session N <short title> plan
```

Files: `SESSION_N_PLAN.md` only.

### Commit 2 — <description>

<Scope. Gates. Files touched.>

### Commit 3 (if needed) — <description>

<Scope. Gates. Files touched.>

---

## H. Gates

- `dart analyze lib/` — must be `0` (or the project's baseline)
- `flutter test` — must pass
- `flutter build <target>` — must succeed
- `flutter gen-l10n` — must exit clean before any code references new keys

---

## I. Hard Constraints

- Do NOT modify any file outside this session's scope.
- Do NOT translate to target languages — ES values are EN placeholders.
- Do NOT touch <prior-session exclusion files>.
- Do NOT speculatively promote keys to `commonX` — 2+ domain evidence required.
- Do NOT skip the push-check at end of session (`git log origin/<branch>..HEAD`).
- <Any session-specific constraints>

---

## J. Follow-up / Deferred

<Items discovered during investigation that are NOT in scope for this session. Noted here so future sessions can pick them up.>

- <item 1>
- <item 2>
