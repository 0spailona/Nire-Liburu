# System Overview

## Context

Nire-Liburu is an Android book reader (txt, fb2, fb2.zip, pdf) with optional
Google sign-in and cross-device reading-progress sync, built serverless entirely on
Google Drive (ADR-001). This overview is the entry point to the project map;
domains, contracts, and decisions elaborate the blocks below.

## Architecture Map

```
┌──────────────────────────────────────────────────────────────────────┐
│                            Android App (Kotlin)                      │
│                                                                      │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐   │
│  │    AUTH    │   │   LIBRARY  │   │   READING  │   │    SYNC    │   │
│  │  domain    │   │   domain   │   │   domain   │   │   domain   │   │
│  │            │   │            │   │            │   │            │   │
│  │ Credential │   │ import,    │   │ txt/fb2    │   │ pull/push  │   │
│  │ Manager +  │   │ metadata,  │   │ paginated  │   │ progress,  │   │
│  │ Google ID  │   │ covers,    │   │ engine +   │   │ upload,    │   │
│  │ token      │   │ delete     │   │ Pdf-       │   │ mode       │   │
│  │            │   │            │   │ Renderer   │   │ upgrade    │   │
│  └─────┬──────┘   └─────┬──────┘   └─────┬──────┘   └─────┬──────┘   │
│        │                │                │                │          │
│        └────────────────┴───────┬────────┴────────────────┘          │
│                                 │                                    │
│                    ┌────────────▼─────────────┐                      │
│                    │  Room DB (source of      │
│                    │  truth on device)        │                      │
│                    │  books · progress ·      │                      │
│                    │  pending sync queue      │                      │
│                    └────────────┬─────────────┘                      │
│                                 │                                    │
│  ┌──────────────────────────────▼───────────────────────────────┐    │
│  │      GOOGLE DRIVE API v3  (contract: drive-storage.md)       │    │
│  │      scope: drive.file (books + progress — ADR-007)          │    │
│  └──────────────────────────────┬───────────────────────────────┘    │
└─────────────────────────────────┼────────────────────────────────────┘
                                  │ HTTPS (OAuth 2.0 token)
                    ┌─────────────▼──────────────┐
                    │   Google Drive (user's)    │
                    │  └─ Nire-Liburu/ (visible) │  ← book files
                    │     └─ progress/*.json     │  ← ADR-007
                    └────────────────────────────┘
```

## Layering

| Layer | Contents | Depends on |
| ----- | -------- | ---------- |
| UI (Compose) | screens: Welcome, Library, Reader (text/pdf), Settings | ViewModels |
| Presentation logic | ViewModels, StateFlow states | domain use-cases |
| Domain | readers, import, sync engine, LWW merge | interfaces only |
| Data | Room entities/DAOs, Drive client wrapper, SAF import | Android platform, Drive API |

Rules:

- UI never touches Drive or Room directly; it observes ViewModels.
- Domain logic (pagination, LWW, hash identity) stays Android-free and
  unit-testable.
- The Drive client is the single implementation of the storage contract
  (`contracts/drive-storage.md`); swapping it must not touch domain logic.

## Primary Flows

**Import (SAF)**: multi-select picker → each file copied into app storage
sequentially → hash → parse metadata → Room insert → (cloud mode) upload to
Drive → record `fileId`. `.fb2.zip` inputs are unpacked at import; identity
and storage use the inner `.fb2`. Duplicates are skipped per file with an
"already in library" notice (ADR-010).

**Open book**: Room lookup → (metadata-only row) size guard on metered
network — ≤ 20 MB silent, > 20 MB asks (ADR-010) → download if needed
(hash-verified) → (cloud) pull progress → LWW vs local → render at
position.

**Reader menu**: always-present floating button (ADR-010) opens the panel:
position indicator (`percent · page x of N`), seek slider (persisted like
a page turn), chapters (fb2 only in v1), reader settings (link to Settings
screen), close book.

**Close book**: persist position to Room → (cloud) enqueue push →
WorkManager writes to `Nire-Liburu/progress/` (retry w/ backoff).

**Mode upgrade (local → cloud)**: sign-in → upload all local books → seed
cloud progress from local (LWW) → switch flag.

**New-device restore (cloud mode)**: `listBooks()` → metadata-only library
rows (covers lazily) → file downloads on first open with hash check and
the metered-size guard (ADR-010).

**Full book deletion (cloud mode)**: long-press on a grid card opens the
deletion dialog; user picks "delete everywhere" → local row + file removed
first → Drive book file deleted
→ progress JSON deleted last; 404 = success; retries via WorkManager
(ADR-008, contract deletion order). "Delete locally only" turns the row
into a metadata-only row (`needsDownload = true`) that re-downloads on
next open.

**Library refresh (cloud mode)**: entering the Library screen or manual
pull-to-refresh runs `listBooks()` and merges Drive files into Room (new
files become metadata-only rows) — the only book-list sync trigger besides
new-device restore (ADR-003 foreground exception).

**Different-account sign-in**: choice dialog — merge local books into the
new account, or keep them local-only (`driveFileId` cleared) and show the
new account's library (ADR-003).

**First launch**: welcome screen — "Sign in with Google" or "Skip — local
mode" (ADR-008); the library (adaptive cover grid) follows either way.

**Settings (ADR-010)**: screen from the Library toolbar with two sections
— Account (email, mode, sign-out — the only sign-out surface) and Reader
(navigation mode).

## Invariants

- The local Room database is the source of truth on the device at all times.
- Reading requires no network in both modes.
- The app performs no background network traffic except WorkManager retries
  of enqueued progress pushes.
- Every book record carries a content-hash `bookId` (SHA-256, 16 hex chars).
- Progress is written to Drive only on book close (push) and read only on
  book open (pull).
- The book list refreshes from Drive only in the foreground: on Library
  screen entry and on manual pull-to-refresh (cloud mode).
- A first-open download over a metered network exceeding 20 MB always asks
  the user; Wi-Fi never asks (ADR-010).
- The SAF import picker always allows multiple selection; each file runs
  through the identical single-file pipeline (ADR-010).
- The app functions fully with no Google account signed in.
- Reading progress is stored only inside the visible `Nire-Liburu/`
  folder (books + `progress/` subfolder); the app never uses
  `appDataFolder` (ADR-007 — it is wiped on app uninstall).
- A full book deletion always removes the Drive book file and the progress
  JSON, in the contract's fixed order, local removal first (ADR-008).
- The library always renders as an adaptive cover grid sorted by
  `lastReadAt` descending, never-read books last (ADR-011); deletion is
  reachable only through card long-press plus the confirmation dialog
  (ADR-011).
- The txt/fb2 reader always offers the configured navigation mode
  (vertical scroll by default) and the fb2 chapter sheet; both are
  presentation-only and never change position semantics (ADR-008).
- The reader always shows the floating menu button; its panel is the only
  home of the position indicator and seek slider, and the seek slider
  never creates a third position kind (ADR-010).
- Sign-out lives only on the Settings screen (ADR-010).
- UI language defaults to Russian (`values/`), with English in
  `values-en/`; no other fallback and no in-app switcher in v1 (ADR-010).
- The app opts out of Android platform backup (`allowBackup=false`); no
  app data — tokens, Room, book files — ever enters Google's backup
  channel. Drive is the only cloud copy (ADR-009).
- The app is locked to portrait orientation in v1 (ADR-009).

## Anti-Patterns

- **UI calling Drive directly** — breaks the single-implementation storage
  contract and testability.
- **Periodic/background full sync** — contradicts ADR-003 and wastes quota.
- **Storing books inside `appDataFolder`** — hides the user's library from
  them (ADR-001 keeps books visible in `Nire-Liburu/`); storing *progress*
  there is likewise forbidden: the folder is wiped on app uninstall
  (ADR-007).
- **Skipping the hash on import** — breaks cross-device progress identity
  (ADR-002).
- **Deleting the Drive copy without the progress JSON (or in reverse
  order)** — leaves an orphan progress file or a half-deleted book; follow
  the contract's fixed deletion order (ADR-008).
- **Adding reader actions outside the floating-button panel** — fragments
  the reader surface; every reader action lives in the panel (ADR-010).
