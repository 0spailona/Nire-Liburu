# Sync Domain

## Purpose

Synchronizes reading progress and book files between the local Room database
and Google Drive, exactly at the open/close sync points (ADR-003), including
deferred retries and the one-time local→cloud mode upgrade.

## Key Files

- `app/src/main/kotlin/…/sync/SyncCoordinator.kt` — orchestrates pull/push,
  invoked by ReaderViewModel at open/close <!-- TODO: verify when code exists
  -->
- `app/src/main/kotlin/…/sync/LwwResolver.kt` — deterministic conflict
  resolution (ADR-002)
- `app/src/main/kotlin/…/sync/ProgressPushWorker.kt` — WorkManager worker
  retrying failed pushes
- `app/src/main/kotlin/…/sync/ModeUpgradeUseCase.kt` — one-time merge on
  sign-in

## Core Types

```kotlin
data class ProgressRecord(          // mirrors progress JSON (ADR-002)
    val schema: Int,                // = 1
    val bookId: String,
    val format: String,
    val positionKind: String,       // "charOffset" | "page"
    val charOffset: Int?,           // txt/fb2
    val page: Int?,                 // pdf
    val percent: Float,
    val deviceId: String,
    val revision: Long,
    val updatedAt: String,          // RFC 3339 UTC
)

data class PendingPush(            // Room queue table
    val bookId: String,
    val payload: ProgressRecord,
    val attempts: Int,
)
```

## Flow

```
OPEN BOOK (cloud mode)
   ▼
read Nire-Liburu/progress/<bookId>.json                (contract)
   ▼
LwwResolver: (local vs remote) → winner → effective position
   local wins  → open at local position
   remote wins → open at remote position (Room updated)

CLOSE BOOK (cloud mode)
   ▼
revision+1, updatedAt=now → Room UPDATE → enqueue PendingPush
   ▼
try files.update → success: clear queue row
                  │ failure (offline / 5xx / rate-limit)
                  ▼
        WorkManager exponential backoff (≤10 attempts, network-constrained)

MODE UPGRADE (sign-in, local mode → cloud)
   ▼
same account as before? ──yes──► resume CLOUD mode (no dialog)
   │ no / first sign-in with local books
   ▼
choice dialog (ADR-003)
   ├─ "upload local books": for each local book: upload to Nire-Liburu/
   │   → save driveFileId; for each local progress: seed cloud via LWW
   │   (only if cloud record absent or local wins)
   └─ "use this account's library": clear driveFileId on local books,
       then pull this account's books via LIBRARY REFRESH

LIBRARY REFRESH (cloud mode; ADR-003 foreground exception)
   ▼
trigger: Library screen entered (cloud mode) or manual pull-to-refresh
   ▼
DriveBooks.listBooks() → merge into Room:
   new Drive files → metadata-only rows (needsDownload=true)
   rows whose local file exists → link driveFileId by content bookId
   metadata-only rows whose Drive file vanished → removed on next open
   (BookFileMissing, silent — contract)

NEW DEVICE (cloud mode, first launch / empty Room)
   ▼
DriveBooks.listBooks() → metadata-only rows (needsDownload=true)
   ▼
library renders placeholders; title/format from Drive file metadata,
cover parsed lazily after download
   ▼
first open of a book: download → streaming hash verifies bookId →
regular OPEN flow (pull progress → LWW → read)

FULL DELETE (cloud mode, "delete everywhere" — ADR-008)
   ▼
order fixed by contract: local row + local file removed first (never
deferred) → Drive book file delete → Nire-Liburu/progress/<bookId>.json
delete; Drive steps retry via WorkManager; 404 = success; progress JSON
is deleted last so an interrupted delete leaves a re-deletable book,
never a progress orphan
```

## Invariants

- Progress is pulled only on book open and pushed only on book close — never
  in the background on a schedule.
- Library *metadata* refresh (`listBooks()`) runs only in the foreground:
  on entering the Library screen (cloud mode) and on manual
  pull-to-refresh — never in the background (ADR-003 exception).
- At most one `PendingPush` row per `bookId`; a newer close replaces an older
  unpushed payload for the same book.
- `revision` strictly increases per book per device across pushes.
- LWW ordering is total and deterministic: `updatedAt` desc → `revision`
  desc → `deviceId` desc.
- A successful push always clears the pending queue row in the same
  transaction that confirms success.
- Mode upgrade is idempotent: running it twice produces no duplicate Drive
  files (same content hash → reuse existing file).
- Progress JSON lives at `Nire-Liburu/progress/<bookId>.json` (visible
  folder, ADR-007); the app never reads or writes `appDataFolder`.
- A full deletion ("delete everywhere", ADR-008) always removes the Drive
  progress JSON after the book file, in the contract's fixed order;
  local removal happens first and is never deferred.

## Configuration

- Backoff: exponential, initial 30 s, factor 2, max attempt count 10,
  network type UNMETERED = false (any network) <!-- TODO: verify values -->.
- WorkManager unique work name per bookId: `progress-push-<bookId>`.

## Extension Points

- Bookmarks/highlights sync: additional files under `Nire-Liburu/progress/`
  (or a sibling subfolder) with their own push workers; open/close triggers
  unchanged.
- Multi-account: scope the sync state by account key.

## Related Specs

- [System Overview](architecture/overview.md) — sync points in flows
- [ADR-002](decisions/002-book-identity-and-progress-model.md) — LWW and
  schema
- [ADR-003](decisions/003-dual-mode-open-close-sync.md) — sync trigger
  policy
- [Drive Storage Contract](contracts/drive-storage.md) — the actual API
  calls
- [Reading Domain](domains/reading/README.md) — trigger source
