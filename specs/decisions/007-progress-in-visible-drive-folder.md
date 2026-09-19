# ADR-007: Progress storage in the visible Drive folder

## Status

Accepted

## Context

ADR-001/002 placed progress JSON files in the hidden `appDataFolder`.
Google's official documentation ("Store application-specific data",
developers.google.com/drive/api/guides/appdata) states that the application
data folder is **deleted when the user uninstalls the app**. The folder is
per-account and shared by all installations of the app: uninstalling the app
on one device (a typical "tried it, removed it" scenario) erases the synced
reading progress for *every* device of that account — contradicting the
core product promise "progress is preserved". The owner chose to move
progress into the visible `Nire-Liburu/` tree.

## Decision

1. Progress JSON files live in a visible subfolder `Nire-Liburu/progress/`,
   one file per book, named `<bookId>.json`. File format, schema, and LWW
   rules are unchanged (ADR-002).
2. The app requests a single OAuth scope: `drive.file`. The `drive.appdata`
   scope is dropped — nothing is stored in `appDataFolder` anymore.
3. `DriveBooks.listBooks()` lists direct children of `Nire-Liburu/` and
   skips folders, so the `progress/` subfolder never appears as a book.
4. No data migration is required: no released build has ever written to
   `appDataFolder` (design-stage decision).

Supersedes in part:

- [ADR-001](./001-drive-only-backend.md) — progress storage location and
  the OAuth scope list; the Drive-only architecture itself stands.
- [ADR-002](./002-book-identity-and-progress-model.md) — storage location
  of the progress files (decision item 3); identity, schema, and LWW
  resolution stand.
- [ADR-003](./003-dual-mode-open-close-sync.md) — the `appDataFolder`
  mentions in the pull/push bullets; sync points and mode semantics stand.
- [ADR-005](./005-technology-stack.md) — the scope note in the Drive table
  row; the technology stack itself stands.

## Consequences

Positive:

- Progress survives app uninstall on any device — the cross-device promise
  holds even if the app is removed and later reinstalled.
- A single non-sensitive scope (`drive.file`) simplifies OAuth consent.
- Progress files are user-visible in the Drive UI: manually copyable as a
  backup, debuggable without API calls.

Negative:

- The user can see, rename, edit, or delete progress files. Deletion maps
  to the existing `NoProgress` fallback (pull falls back to the local
  position); an unparsable JSON is treated the same way (contract error
  table).
- The `progress/` subfolder is visible inside the user's `Nire-Liburu/`
  folder in the Drive UI.

## Alternatives Considered

- **Keep `appDataFolder`** — rejected by owner decision: uninstalling the
  app on any device wipes progress for all devices; unacceptable for the
  product promise.
- **Dual-write (`appDataFolder` + visible mirror)** — rejected: two writes
  per push, double request quota, and mirror-merge logic with no benefit
  over one visible location.
- **Firestore for progress** — already rejected in ADR-001; unchanged.
