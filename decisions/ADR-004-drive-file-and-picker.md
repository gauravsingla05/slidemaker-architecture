# Note 4 — `drive.file` + Google Picker over `spreadsheets.readonly`

**Status:** Accepted
**Context chapter:** [7. OAuth & token storage](../07-oauth-and-token-vault.md)

## Context

The scheduled-decks Google Sheets data source needs to read a single
spreadsheet on a recurring schedule. The intuitive scope is
`spreadsheets.readonly`, which grants read access to every
spreadsheet in the user's Drive.

Google classifies `spreadsheets.readonly` as a **sensitive** scope.
An application requesting it must pass Google's verification process,
including a security review. Verification is calendar-time expensive.

## Decision

Use `drive.file` instead, combined with the Google Picker UI for file
selection. `drive.file` is a **non-sensitive** scope; it grants
access only to files the user explicitly picks via Picker (or files
the application creates).

```
   spreadsheets.readonly (sensitive)
   ---------------------------------
     User consents to read ALL sheets
                 |
                 v
     App lists user's sheets
                 |
                 v
     User picks one


   drive.file + Picker (non-sensitive)
   -----------------------------------
     User consents to per-file access
                 |
                 v
     User picks via Picker
       (or pastes URL)
                 |
                 v
     App receives file id
```

## Alternatives considered

| Option | Pros | Cons | Rejected because |
|--------|------|------|------------------|
| `spreadsheets.readonly` | Single consent | Sensitive scope → full Google verification | Verification cost too high |
| Service account + per-file share | No user consent for the app | User must share each sheet with a service account email; confusing UX | Bad UX |
| Export sheet to CSV upload | No OAuth at all | Defeats the "live data" value of a recurring schedule | Wrong feature |
| **drive.file + Picker (chosen)** | Non-sensitive scope, per-file consent surfaces the privacy model | Picker quirk: can only list previously-granted files | Best trade-off |

## The Picker quirk and the workaround

`drive.file` only lets the Picker list files the application
*already* has access to. For a new user that list is empty — the
Picker appears to show no spreadsheets at all.

Workaround: collect the sheet's URL from the user via a paste field,
then call `setFileIds([sheetId])` on the Picker. The Picker now
pre-navigates to that specific file; selecting it grants access.
Subsequent runs read the file by id without further user
interaction.

## Consequences

- A schedule that references a Google Sheet requires the user to
  confirm *that specific sheet* once. Subsequent runs are silent.
- The flow is two steps (paste URL → confirm in Picker) instead of
  one (browse and pick), but stays in the non-sensitive scope band.
- No Google security review is required to ship this feature.
- If the user later un-shares the sheet, runs fail cleanly and the
  schedule auto-pauses after enough consecutive failures.

## Revisit when

- Verification investment becomes acceptable and the broader-scope
  UX unlocks meaningful value.
- Google relaxes Picker's `drive.file` constraints.
