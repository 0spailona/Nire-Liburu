# Library Domain

## Purpose

Owns the book catalog: importing files from device storage (SAF), extracting
metadata and covers, keeping Room records in sync with Drive in Cloud mode,
and presenting the library as a cover grid.

## Key Files

- `app/src/main/kotlin/…/library/LibraryViewModel.kt` — library screen
  state <!-- TODO: verify when code exists -->
- `app/src/main/kotlin/…/library/ImportBookUseCase.kt` — SAF uri → app
  storage copy → hash → parse → Room insert
- `app/src/main/kotlin/…/library/Fb2MetadataParser.kt` — title-info
  extraction (title, author, cover href)
- `app/src/main/kotlin/…/library/BookRepository.kt` — Room CRUD + Drive
  linkage (`driveFileId`)

## Core Types

```kotlin
enum class BookFormat { TXT, FB2, PDF }

data class Book(
    val bookId: String,        // SHA-256, first 16 hex chars
    val title: String,
    val author: String?,
    val format: BookFormat,
    val filePath: String,      // app-internal storage
    val fileSizeBytes: Long,
    val coverPath: String?,    // fb2 covers; null otherwise
    val driveFileId: String?,  // null until uploaded (Cloud mode)
    val needsDownload: Boolean = false, // cloud-restore row, file not local
    val addedAt: Instant,
    val lastReadAt: Instant?,
)
```

## Flow

```
SAF picker (multi-select: txt/fb2/fb2.zip/pdf — ADR-010)
   │ each selected file is imported sequentially through the
   │ single-file pipeline below (duplicates skipped with a per-file
   │ "already in library" notice)
   ▼
copy to app storage (no storage permissions needed)
   ▼
.fb2.zip? ──yes──► unpack single inner .fb2 (hash/upload/store inner bytes)
   │ no
   ▼
streaming SHA-256 → bookId
   ▼
detect format (content signature > extension)          [ADR-004]
   ▼
parse metadata: fb2 title/author/cover · txt/pdf → filename
   ▼
Room: INSERT book (driveFileId = null)
   │
   ├─ duplicate bookId already in Room? ──► no-op + notice "already in
   │   library"; reading position untouched (ADR-006) ──► stop
   ▼
Cloud mode only: upload → Nire-Liburu/<name>  →  UPDATE driveFileId
                  (contract: drive-storage.md)

new device, cloud mode: DriveBooks.listBooks() → metadata-only rows
   (needsDownload=true, cover fetched lazily) → file downloads on first
   open (streaming hash verifies identity)              [ADR-003]

first-open download guard (ADR-010):
   metered network? ──no──► download immediately
   │ yes
   Drive file size ≤ 20 MB? ──yes──► download silently
   │ no (> 20 MB)
   confirmation dialog "book is X MB — download over mobile data?"
      ├─ confirm ──► download
      └─ decline ──► stay on library; row remains metadata-only;
                      next open re-prompts

library refresh (cloud mode): triggered on Library screen entry and on
   manual pull-to-refresh (ADR-003); listBooks() merges Drive files into
   Room — new files become metadata-only rows; rows already linked keep
   their local files

delete book ──► confirmation dialog (ADR-008):
   ├─ book has driveFileId (Cloud mode):
   │    ├─ "delete everywhere": Room DELETE + local file first, then
   │    │   Drive book file, then Nire-Liburu/progress/<bookId>.json
   │    │   (contract deletion order; 404 = success; WorkManager retry)
   │    └─ "delete locally only": local file removed, row becomes
   │        metadata-only (needsDownload = true) and stays in library;
   │        Drive copy and progress JSON remain; re-download on next open
   └─ book has no driveFileId (LOCAL mode / local-only books):
        single destructive action: Room DELETE + local file delete
        (nothing exists in the cloud)
```

## Invariants

- Every book record has a non-null `bookId` computed from file content.
- Importing a book whose `bookId` already exists in Room is a no-op with an
  "already in library" notice; position and metadata are never overwritten
  (ADR-006).
- The SAF picker always allows multiple selection; every selected file
  passes through the identical single-file pipeline in sequence (ADR-010).
- A first-open download on a metered network over 20 MB always asks the
  user first; Wi-Fi downloads never ask; a declined download leaves the
  row metadata-only and re-prompts on the next open (ADR-010).
- Imported files live in app-internal storage; originals are never
  modified.
- Import works identically in LOCAL and CLOUD modes; upload is an additive
  follow-up step.
- The library renders an adaptive cover grid (2–3 columns by screen width),
  sorted by `lastReadAt` descending (most recent first), never-read books
  (null `lastReadAt`) always after read ones (ADR-011).
- Deleting a book always goes through long-press on the grid card and the
  ADR-008 confirmation dialog; tap always opens the book, and no other
  delete surface exists (ADR-011). In Cloud mode the dialog
  distinguishes "delete everywhere" from "delete locally only", and a full
  deletion always removes the Drive progress JSON together with the book
  file (ADR-008).
- Local removal in a full deletion always happens first and is never
  deferred behind Drive operations (contract deletion order).
- Display names are not user-editable in v1: the title comes from fb2
  metadata or the imported file name; no rename/alias surface exists
  (ADR-009).

## Configuration

- Supported import extensions/MIME: `.txt` (`text/plain`), `.fb2`
  (`application/x-fictionbook`, `text/xml`), `.fb2.zip` (zip container with
  a single inner `.fb2`, unpacked at import — ADR-004), `.pdf`
  (`application/pdf`).
- SAF picker mode: multi-select (ADR-010).
- Metered-download threshold: 20 MB (ADR-010) <!-- TODO: verify constant
  value -->.
- Cover thumbnail size: 256 px longest side <!-- TODO: verify -->.

## Extension Points

- Future collections/tags: new table keyed by `bookId`, no changes to
  existing flows.
- EPUB: add parser + format enum value; import pipeline unchanged.
- Drive→device import (pick existing `Nire-Liburu/` files on a new device):
  reuses the same hash identity, adds a download branch.

## Related Specs

- [System Overview](architecture/overview.md) — import flow context
- [ADR-002](decisions/002-book-identity-and-progress-model.md) — bookId
- [ADR-004](decisions/004-formats-and-pdf-strategy.md) — format detection
- [Reading Domain](domains/reading/README.md) — consumers of book files
- [Sync Domain](domains/sync/README.md) — upload orchestration
