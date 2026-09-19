# Contract: App <-> Google Drive

## Boundary Rule

The Drive client wrapper is the ONLY component allowed to talk to the Google
Drive API; domain and UI code depend on the interfaces below, never on Drive
types.

## Interfaces

| Interface | Package | Consumed By | Purpose |
| --------- | ------- | ----------- | ------- |
| `DriveBooks.findRoot(): DriveFile?` / `ensureRoot()` | `data.drive` | Library, Sync | locate or create the `Nire-Liburu/` root folder |
| `DriveBooks.upload(file, name): DriveFile` | `data.drive` | Library, Sync | upload a book file (resumable) |
| `DriveBooks.download(fileId): File` | `data.drive` | Library | download book to app storage |
| `DriveBooks.delete(fileId)` | `data.drive` | Library | delete book from Drive |
| `DriveBooks.listBooks(): List<DriveFile>` | `data.drive` | Sync (mode upgrade, new-device pull, library refresh) | list direct children of `Nire-Liburu/`, folders skipped |
| `DriveProgress.read(bookId): ProgressRecord?` | `data.drive` | Sync | fetch `progress/<bookId>.json` from `Nire-Liburu/progress/` |
| `DriveProgress.write(bookId, record)` | `data.drive` | Sync (push worker) | create/update the progress file |
| `DriveProgress.delete(bookId)` | `data.drive` | Library (full deletion) | delete `progress/<bookId>.json` (ADR-008) |
| `TokenProvider.token(): String` | `auth` | data.drive | valid access token, silent refresh |

## Initialization

- All Drive calls require an OAuth access token with scope `drive.file`,
  obtained via `TokenProvider` (auth domain). No other scope is requested
  (ADR-007).
- `ensureRoot()` runs once per cloud session (first Drive operation):
  `files.list(q: name='Nire-Liburu' and mimeType=folder and trashed=false,
  spaces: drive)` → create via `files.create` if absent. The returned
  `rootFolderId` is cached in Room. `ensureProgressFolder()` locates or
  creates `Nire-Liburu/progress/` the same way, before the first
  `DriveProgress` call.

## Data Flow Across Boundary

**Drive layout (the contract of "what lives where"):**

```
My Drive (user-visible)
└── Nire-Liburu/                     ← root folder (mimeType: folder)
    ├── book-name.fb2                ← book files, original bytes preserved
    ├── book-name.pdf
    ├── book-name.txt
    └── progress/                    ← subfolder (ADR-007)
        ├── 0f2a9c1d2e3b4f56.json    ← one file per bookId
        └── ...
```

Everything lives inside `Nire-Liburu/`; nothing is stored in
`appDataFolder` (ADR-007: the app-data folder is wiped on app uninstall,
which would erase progress for every device of the account).

**In → out (reads)**: `DriveProgress.read` → `files.list(spaces=drive,
q: name='<bookId>.json' in parents=<progressFolderId>)` →
`files.get(alt=media)` → kotlinx.serialization → `ProgressRecord`.

**Out → in (writes)**: `ProgressRecord` → JSON (`schema=1`, ADR-002 fields)
→ `files.update` (or `files.create` with `parents=[progressFolderId]` on
first write). File media type: `application/json`.

**Deletion order (full delete, ADR-008)**: the local row and local file are
removed first (immediate, never deferred); then the Drive book file
(`DriveBooks.delete`), then the progress JSON (`DriveProgress.delete`).
Drive deletions run through WorkManager retry on failure; HTTP 404 counts
as success (the file is already gone). The progress JSON is deleted last,
so an interrupted full delete leaves a deletable book rather than a
progress-orphan.

**Books**: uploaded with `files.create` (resumable for files > 5 MB),
`parents=[rootFolderId]`, name = original filename. Downloads stream to app
storage; the local copy's hash re-derives `bookId` (identity check).

## Error Propagation

| Drive error | Mapped to | Handling |
| ----------- | --------- | -------- |
| 401 / invalid token | `AuthError` | silent re-auth via `TokenProvider`, one retry, then sign-out prompt |
| 403 rate limit / `userRateLimitExceeded` | `RetryableSyncError` | WorkManager backoff |
| 5xx / IOException | `RetryableSyncError` | WorkManager backoff |
| 404 progress file missing | `NoProgress` (not an error) | pull falls back to local position |
| progress JSON unparsable / unreadable | `NoProgress` (not an error) | pull falls back to local position; the file is overwritten on next push (ADR-007 consequence) |
| 404 book file missing (download) | `BookFileMissing` | remove the metadata-only library row silently (owner's choice); the progress JSON stays in `Nire-Liburu/progress/` |
| 416/other 4xx | `PermanentSyncError` | surface to user, drop pending row after N attempts |

- Pull failures never block opening a book: local position is used.
- `BookFileMissing` never deletes the user's local copy — only metadata-only
  rows (`needsDownload = true`, no local file) are removed; a book with a
  local file whose Drive copy vanished stays fully usable offline.
- A progress-JSON 404 during full deletion counts as success (ADR-008).
- All errors cross the boundary as sealed types (`SyncError` hierarchy);
  Drive/HTTP details never leak into domain code.

## Breaking Change Checklist

- If the progress JSON gains a field → bump `schema`, keep v1 readers
  tolerant (unknown fields ignored), update [ADR-002](decisions/002-book-identity-and-progress-model.md)
  and the Sync domain spec.
- If the Drive folder name changes → write a migration (list old name, move
  files), update this contract and the Library domain spec.
- If a new scope is requested → update [ADR-001](decisions/001-drive-only-backend.md),
  the auth domain spec, and OAuth consent configuration.
- If books move out of `drive.file` scope (e.g. full `drive` scope) → new ADR
  required (consent implications).
- If the progress location moves again → new ADR (as ADR-007 did) plus a
  data migration path (read old location, write new, delete old).
