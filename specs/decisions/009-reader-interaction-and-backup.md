# ADR-009: Reader interaction refinements and backup opt-out

## Status

Accepted

## Context

Six product decisions were made by the owner after ADR-008: (1) the app is
locked to portrait orientation in v1 — avoiding Activity recreation and
re-pagination on rotation entirely; (2) full-text search in txt/fb2 stays
out of v1; (3) the PDF reader gets pinch-to-zoom — without it, small text
is often unreadable on a phone, and `PdfRenderer` renders bitmaps so zoom
is cheap; (4) the app opts out of Android cloud backup entirely
(`allowBackup=false`) — auth tokens must never leave the device, and
progress already has its own cloud channel (Drive, ADR-007); (5) book
renaming stays out of v1 — the title comes from the file name / fb2
metadata; (6) text selection/copy in the txt/fb2 reader stays out of v1
(distinct from the deferred highlights feature — this covers even plain
system copy).

## Decision

1. **Portrait lock.** The whole app (not just the reader) is locked to
   portrait in v1 via the manifest
   (`android:screenOrientation="portrait"`). No code path has to handle
   configuration-driven recreation of the reader or re-pagination.
   Landscape support is an extension point, not a v1 surface.
2. **No full-text search in v1.** Neither the txt/fb2 engine nor the
   library exposes text search. Adding it later requires no position-model
   change (results map to `charOffset`).
3. **PDF pinch-to-zoom in v1.** The PDF page bitmap supports pinch-zoom
   and pan within the page. Zoom is presentation-only: it never changes
   the `page` progress kind (ADR-004), is not persisted, and resets to
   fit-width on page turn. Tap zones keep working (a tap is distinguished
   from a pinch/drag gesture).
4. **Android backup opt-out.** `android:allowBackup="false"` plus
   `dataExtractionRules` disabling cloud backup and device-to-device
   transfer for API 31+. Nothing from app storage enters Google's
   platform backup: no auth tokens, no Room DB, no downloaded book files.
   Drive (ADR-001/007) remains the only cloud copy of books and progress;
   a lost device loses only local-only (LOCAL-mode) data.
5. **No renaming in v1.** Book display names come from fb2 metadata
   (title-author) or the imported file name; no user-editable alias.
6. **No text selection/copy in v1.** The txt/fb2 reader renders plain,
   non-selectable text; long-press has no copy behavior. (Highlights and
   notes remain deferred per ADR-006.)

Supersedes in part:

- [ADR-006](./006-v1-scope-localization-distribution.md) — the v1
  boundary additionally includes the PDF pinch-zoom and the portrait lock;
  the out-of-scope list additionally names full-text search, book
  renaming, and text selection/copy.

## Consequences

Positive:

- Portrait lock removes the whole class of rotation/`configChanges` bugs
  from the reader (state loss, double pagination, position jumps).
- PDF zoom makes dense PDFs actually readable on a phone at near-zero
  architectural cost (bitmap scale + pan).
- Backup opt-out keeps OAuth tokens strictly on-device and makes the
  privacy story simple: one cloud (the user's own Drive), one channel.
- No search/rename/selection keeps the reader surface minimal and the v1
  beta small.

Negative:

- Landscape users (tablets, accessibility preferences) must wait for a
  post-v1 release; the manifest lock is a single-line change to reverse.
- A user who loses a device in LOCAL mode loses everything — the welcome
  screen and library empty-state should nudge toward Cloud mode.
- Re-importing after opt-out restores books but not local-only reading
  positions (only Cloud-mode positions survive via Drive).

## Alternatives Considered

- **Handle rotation with state preservation** — rejected for v1: costs
  config-change plumbing across reader, paginator, and ViewModels for a
  marginal benefit; can be added post-v1.
- **Selective backup (backup rules excluding tokens)** — rejected: backup
  rules are easy to get wrong, and progress already syncs through Drive —
  a second cloud copy adds confusion, not safety.
- **Fit-width only (no PDF zoom)** — rejected by owner decision: zoom is
  in.
- **Search/renaming/selection in v1** — rejected by owner decision: out of
  scope.
