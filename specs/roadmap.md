# Implementation Roadmap

Status: Approved (design), execution gated by explicit owner command

This file is the execution plan for building the Nire-Liburu v1 Android app
from the specs in this repository. It does not change any decision: every task
cites the ADR or domain spec that governs it. Execution does not start until
the owner explicitly commands it.

Sources of truth (read order for any task): [INDEX.md](./INDEX.md) →
[architecture/overview.md](./architecture/overview.md) → the cited domain/ADR.

## Phases

- [ ] Phase 0 — Project bootstrap
- [ ] Phase 1 — Data layer (models, Room, repositories)
- [ ] Phase 2 — Formats: txt + fb2 parsing and pagination engine
- [ ] Phase 3 — Library: import, grid, delete
- [ ] Phase 4 — Reader: rendering, navigation, menu
- [ ] Phase 5 — Cloud: Google sign-in and Drive sync
- [ ] Phase 6 — Settings, welcome, localization
- [ ] Phase 7 — Beta hardening and APK

Dependency spine: 0 → 1 → {2, 3-data} → 3-ui → 4 → 5 → 6 → 7.
Phase 2 and the data half of Phase 3 can run in parallel after Phase 1.

---

## Phase 0 — Project bootstrap

### T0.1 Create Android project skeleton

- What: Empty Compose project with the stack fixed by ADR-005.
- How: Gradle Kotlin DSL, version catalog; applicationId `com.nireliburu`;
  minSdk 26, target/compile per ADR-005; Kotlin, Compose + Material 3,
  Hilt, Room, WorkManager, kotlinx.serialization, Credential Manager;
  portrait-lock on the single Activity (ADR-009 p.1);
  `allowBackup=false` + `dataExtractionRules` excluding cloud backup and
  D2D transfer (ADR-009 p.4); RU as default `values/`, EN in `values-en/`
  (ADR-010 p.3); light theme only (ADR-006).
- Where: repo root (new Android project; this specs/ tree stays authoritative).
- Acceptance criteria:
  - `./gradlew assembleDebug` succeeds on a clean checkout.
  - App installs and shows an empty Activity; rotation is locked to portrait.
  - `android:allowBackup="false"` present; backup rules forbid all transfer.
  - Dependency set matches ADR-005 exactly (no extra frameworks).

### T0.2 Resolve design-stage decision points for this phase

- What: Close the decision points that block code in later phases.
- How: Decide and record (as code docs or new ADRs, never by editing
  superseded ADRs): Drive API client for Kotlin coroutines (REST via Retrofit
  vs google-api-services-drive), OAuth client id mechanism (debug/release),
  retry/backoff parameter values for the sync queue, PDF page cache budget,
  cover image size cap, mobile-data guard constant (ADR-010 p.5, nominal 20 MB),
  ordering rule inside the never-read group of the library sort (ADR-011).
- Where: new short ADRs in [decisions/](./decisions/) following
  [\_template.md](./decisions/_template.md) when the choice is architectural;
  code-level constants otherwise.
- Acceptance criteria:
  - Every decision point listed above has a written decision with a rationale
    and a citation of the ADR it supports.
  - No decision contradicts [contracts/drive-storage.md](./contracts/drive-storage.md)
    or ADR-007 (single scope `drive.file`, progress in visible folder).

---

## Phase 1 — Data layer

### T1.1 Book identity and progress model

- What: Types and pure functions for bookId and progress documents.
- How: `bookId = sha256(content)[0..16)` as 16 lowercase hex chars (ADR-002);
  progress JSON schema v1 exactly as in
  [contracts/drive-storage.md](./contracts/drive-storage.md); LWW comparison
  (updatedAt, then revision, then deviceId) as a pure function with unit tests;
  position types: `charOffset` for txt/fb2, `page` for pdf — never a third kind.
- Where: new `core/model` + `core/identity` packages.
- Acceptance criteria:
  - Unit tests: identical bytes → identical bookId across runs and devices;
    fb2 and fb2.zip of the same content produce the same bookId after unzip
    (ADR-004 unzip-at-import rule).
  - LWW tests cover all tie-break branches and timezone/clock-skew inputs.
  - kotlinx.serialization round-trips the progress JSON byte-exactly
    (golden-file test).

### T1.2 Room schema and repositories

- What: Local persistence for books and pending sync operations.
- How: Entities `BookEntity` (id, displayName, author, format, addedAt,
  lastReadAt, needsDownload, driveFileId nullable, localUri nullable),
  `PendingOpEntity` (sync queue rows for WorkManager). DAOs behind repository
  interfaces; no Google/Drive imports allowed in this layer (overview
  anti-pattern rule). Queries for: library grid sorted by lastReadAt desc with
  null-lastReadAt always last (ADR-011), books needing download, pending ops
  by bookId.
- Where: `data/local` package; repositories in `data/repo`.
- Acceptance criteria:
  - Instrumented Room tests: sort order invariant holds (null-last always
    after read ones, stable within groups per T0.2 decision).
  - Repository interfaces compile without any Drive/network dependency.
  - Metadata-only card (`needsDownload=true`, `localUri=null`) round-trips.

---

## Phase 2 — Formats: txt + fb2 engine

### T2.1 Parsers (txt encoding detection, fb2 XML)

- What: Byte stream → text, and fb2 → structured body + metadata.
- How: txt: BOM → UTF-8 → windows-1251 auto-detection (ADR-004); fb2: XML pull
  parse of body/section into a flat chapter list (title + charOffset range,
  ADR-008 TOC-in-v1) plus title/authors/cover for metadata; fb2.zip: accept
  exactly one entry, unzip at import, error messages for 0 or 2+ entries
  (ADR-004).
- Where: `formats/txt`, `formats/fb2` packages.
- Acceptance criteria:
  - Golden-file tests: UTF-8 with/without BOM, UTF-16 BOMs, windows-1251
    sample; wrong-encoding garbage produces a clear error, not mojibake.
  - fb2 fixture with 3 sections yields flat TOC with correct char offsets.
  - Zip with 1 entry OK; 0 and 2+ entries produce the documented errors.

### T2.2 Pagination engine (deterministic reflow)

- What: Same-viewport → same pages, charOffset ↔ page mapping.
- How: Measure text into pages at a given viewport (font size fixed in v1,
  ADR-006); pure, testable core separate from Compose; cache per bookId +
  viewport signature; slider seek maps target page → charOffset and back
  (ADR-010 p.6); NavMode is presentation-only: VERTICAL_SCROLL (default) and
  HORIZONTAL_PAGING consume the same pages (ADR-008).
- Where: `formats/pagination` package.
- Acceptance criteria:
  - Property test: paginate(text, viewport) is deterministic — identical
    inputs give identical page boundaries (spec: identical inputs → identical
    pagination).
  - Round-trip test: charOffset → page → charOffset returns the same offset.
  - Performance: pagination of a 2 MB txt on a mid-range viewport completes
    under a locally documented budget; engine has no Android UI imports.

### T2.3 PDF rendering path

- What: Platform PdfRenderer behind the same position model.
- How: Page bitmaps at fit-width; pinch-zoom + pan is view-only, not
  persisted, resets to fit-width on page turn (ADR-009 p.3); progress = page
  number; tap zones identical to text formats; no PDF TOC in v1 (ADR-010 p.7);
  LRU bitmap cache bounded by the T0.2 budget.
- Where: `formats/pdf` package.
- Acceptance criteria:
  - Fixture PDF renders all pages; progress persists as page and survives
    process death.
  - Cache never exceeds the documented budget under a stress test.
  - No pdf-specific position type leaks outside `formats/pdf`.

---

## Phase 3 — Library

### T3.1 Import pipeline (SAF, multi-select)

- What: File picking → hash → metadata → Room, one shared pipeline.
- How: `ACTION_OPEN_DOCUMENT` with multi-select (ADR-010 p.4); per file:
  persist via SAF, sha256 → bookId, extract metadata (fb2) or filename
  (txt/pdf), unzip fb2.zip first; duplicate (same bookId) → no-op with a
  per-file notice, position untouched (ADR-006); no storage permissions —
  SAF only (ADR-006); no rename support (ADR-009 p.5).
- Where: `library/import` package + domain README flows.
- Acceptance criteria:
  - Import of a mixed multi-selection (txt + fb2 + fb2.zip + pdf) creates all
    cards with correct metadata; failure of one file does not abort others.
  - Re-importing an existing book leaves lastReadAt and progress untouched
    and shows the duplicate notice.
  - App requests no storage runtime permissions.

### T3.2 Library grid UI

- What: Responsive cover grid with sort and delete entry.
- How: Adaptive 2–3 column grid (ADR-008): cover or format-styled placeholder,
  title, author, read percent; long-press → ADR-008 delete dialog
  ("everywhere" / "local only" / single action for local-only books);
  tap always opens the book; no icons or multi-select on cards (ADR-011);
  pull-to-refresh and auto `listBooks()` refresh in cloud mode (ADR-003
  foreground exception).
- Where: `library/ui` package.
- Acceptance criteria:
  - Grid recomposes correctly from Room flows; percent matches progress docs.
  - Long-press on cloud book shows the two-option dialog; on local-only book
    a single delete; tap never deletes.
  - UI tests (Compose): sorting invariant visible; metadata-only card renders
    a placeholder and download state.

---

## Phase 4 — Reader

### T4.1 Reader screens for txt/fb2 and pdf

- What: One reader shell, two renderers, identical chrome and gestures.
- How: Tap zones (left third back, right two thirds forward) everywhere
  including zoomed PDF (ADR-006/ADR-009 p.3); NavMode from settings, default
  VERTICAL_SCROLL (ADR-008); immersive fullscreen, panels via gesture;
  `FLAG_KEEP_SCREEN_ON` (ADR-006); no text selection, long-press inert
  (ADR-009 p.6); restore exact position on reopen.
- Where: `reader/ui` package.
- Acceptance criteria:
  - Reopen after process death restores the same position in all three
    formats and both NavModes.
  - Screenshot tests: immersive state, both NavModes, placeholder covers.
  - Long-press produces no selection and no menu in any format.

### T4.2 Floating menu, TOC, slider

- What: Always-visible translucent menu button; panel with position,
  slider, chapters, settings, close.
- How: Per ADR-010 p.1/p.6 and ADR-008: panel shows "percent · page x of N"
  (pdf: page x of N); slider maps to charOffset/page, release persists the
  position exactly like a page turn; flat fb2 chapter list jumps by
  charOffset; close returns to library with position saved.
- Where: `reader/ui/menu` package.
- Acceptance criteria:
  - Slider release persists a position that survives reopen.
  - Chapter tap lands at the chapter start offset (fb2); menu entries
    disabled for pdf TOC with no dead UI.
  - Menu button does not intercept tap-zone events outside its bounds.

---

## Phase 5 — Cloud: auth + Drive sync

### T5.1 Google sign-in, modes, settings account section

- What: Credential Manager sign-in; LOCAL/CLOUD modes; sign-out.
- How: ADR-001/ADR-005/ADR-010 p.2: single Drive scope `drive.file` only
  (ADR-007); welcome screen on first launch with "Sign in with Google" /
  "Skip — local mode" (ADR-008); Settings screen from library toolbar with
  Account section (email, mode, sign-out) and Reader section (NavMode) —
  the only sign-out entry point.
- Where: `auth/` package; screens per domain README.
- Acceptance criteria:
  - Consent shows exactly the Drive file scope; no appdata scope requested
    anywhere in code.
  - Sign-out returns to welcome; re-sign-in same account does not show the
    account-change dialog (ADR-003).
  - Sign-in with a different account offers merge-upload vs this-account
    library, and local books become local-only with `driveFileId` cleared in
    the second branch (ADR-003; overview account-switch flow).

### T5.2 Drive storage adapter

- What: The single module allowed to call the Drive API.
- How: Implement [contracts/drive-storage.md](./contracts/drive-storage.md)
  exactly: visible folder `Nire-Liburu/` for books, `Nire-Liburu/progress/`
  for progress JSON (ADR-007); `listBooks()` ignores the progress folder;
  interfaces DriveBooks/DriveProgress incl. `delete(bookId)`; Drive errors →
  sync error types per the contract mapping table; mobile-data guard:
  silent ≤ limit, confirm dialog above it on metered, Wi-Fi never asks,
  refusal leaves a metadata-only card (ADR-010 p.5).
- Where: `sync/drive` package (the only package importing Drive types).
- Acceptance criteria:
  - Contract test suite passes against the Drive test double AND a live
    smoke account: upload/list/download/delete for books and progress.
  - Error mapping table is exhaustively unit-tested.
  - `listBooks()` never returns progress JSON files; metered-network dialog
    logic is unit-tested for boundary values from T0.2.

### T5.3 Open/close sync, WorkManager queue, mode upgrade

- What: Pull on open, push on close, durable retry, one-time merge.
- How: ADR-003 flows incl. NEW DEVICE (metadata cards immediately, download
  on first open with hash verification), pull-to-refresh/library-enter
  refresh, WorkManager retry with backoff from T0.2; manual file deletion in
  Drive → silent metadata-only removal (`BookFileMissing`); full delete order
  room+file → drive file → drive progress, 404 = success (ADR-008 +
  contract); one-time merge when upgrading local → cloud (ADR-003).
- Where: `sync/` package; workers registered at app level.
- Acceptance criteria:
  - Integration test with a fake Drive: offline close enqueues, worker
    retries, converges without duplicates; LWW picks the newer doc in both
    directions.
  - New-device flow: cards appear without file download; first open downloads
    once (no re-download on second open).
  - Deleting a Drive file manually externally then refreshing removes only
    the card; local file never deleted by the app.
  - Full-delete interruption test: killing the app mid-order leaves a state
    the worker completes on next run (idempotent order per contract).

---

## Phase 6 — Settings, welcome, localization

### T6.1 Settings and welcome screens finalized

- What: Full Settings screen and first-run welcome per ADR-008/ADR-010.
- How: Settings sections Account + Reader (NavMode radio); welcome once per
  sign-out cycle; empty-state and welcome copy mention long-press delete
  (ADR-011 known drawback); long-press discoverability text lives in strings.
- Where: `settings/`, `onboarding/` packages.
- Acceptance criteria:
  - Fresh install shows welcome exactly once before library; after sign-out
    it shows again.
  - Changing NavMode takes effect on next reader open without restart.
  - Every string is in resources; no hardcoded user-visible text.

### T6.2 RU/EN completeness pass

- What: Default RU, full EN, third locale falls back to RU.
- How: `values/` = RU, `values-en/` = EN (ADR-010 p.3); lint check for missing
  translations; formats/plurals audited (pages, percent).
- Where: `app/src/main/res`.
- Acceptance criteria:
  - `./gradlew lint` reports no missing-translation errors.
  - Device in a third locale (e.g. de) shows Russian strings, not keys.

---

## Phase 7 — Beta hardening

### T7.1 Failure-mode sweep and invariants audit

- What: Walk every invariant in the specs and add a test or a documented
  reason it cannot fail.
- How: Checklist source: invariants sections of
  [architecture/overview.md](./architecture/overview.md) and the four domain
  READMEs; include: never a third position kind; sync never in background
  except library-enter/refresh; no storage permissions; no backup of anything;
  single Drive package boundary (architecture anti-patterns).
- Where: test sources; findings appended to this file's changelog section.
- Acceptance criteria:
  - Each invariant has a named test or a written justification in the PR
    description; zero unchecked items in the audit table.

### T7.2 Release beta APK

- What: Signed beta build distributed for testing.
- How: Per ADR-006: beta APK, both scopes non-sensitive so no strict
  verification needed; proguard rules for kotlinx.serialization and Hilt;
  versioning beta scheme.
- Where: CI config + release profile.
- Acceptance criteria:
  - `./gradlew assembleRelease` produces a signed APK that installs and passes
    a manual smoke: import → read → close (sync) → reinstall → new-device
    restore → progress intact.
  - No debuggable flag in release; no cleartext traffic.

---

## Execution rules

1. One task per branch; acceptance criteria are the definition of done.
2. Never edit an accepted ADR to fit code; write a new ADR and mark the old
   one partially superseded (WORKFLOW rule).
3. Every task's "Where" paths are created fresh — no code exists yet; specs
   remain authoritative when code and spec disagree (fix the code).
4. Decision points from T0.2 must be resolved before the phase that consumes
   them; record outcomes in new ADRs or documented constants.
