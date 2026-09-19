# ADR-008: v1 scope revision — TOC, navigation modes, grid, welcome, deletion

## Status

Accepted (partially superseded by
[ADR-009](./009-reader-interaction-and-backup.md): PDF pinch-zoom added
to v1; portrait lock, backup opt-out, and the search/rename/selection
exclusions settled there; partially superseded by
[ADR-010](./010-reader-chrome-settings-language-ux.md): the reader
navigation-mode setting now lives on the Settings screen Reader section;
the reader gained the floating menu button, position indicator, and seek
slider; partially superseded by
[ADR-011](./011-library-grid-sort-and-delete-entry.md): the grid's
never-read sort placement is confirmed and the delete entry point is the
card long-press)

## Context

Five product decisions were made by the owner after ADR-006 fixed the v1
boundary: (1) the fb2 table of contents belongs in v1 — ADR-004 already
promised "chapter navigation" while ADR-006 left it out, a contradiction;
(2) reader navigation supports tap plus a scroll-mode setting (vertical by
default); (3) the library is a cover grid, not a list; (4) first launch
shows a welcome screen; (5) book deletion offers "delete everywhere" vs
"delete locally only", and a full delete also removes the progress JSON
from Drive.

## Decision

1. **fb2 TOC in v1.** The reader shows a chapter sheet built from the
   `body/section` titles already extracted by the Fb2Parser; selecting a
   chapter jumps to its `charOffset` in the flattened text. No nested
   sub-section tree in v1 — a flat chapter list.
2. **Navigation modes (txt/fb2 reader).** Two modes, selectable in reader
   settings: *vertical scroll* (continuous, default) and *horizontal
   paging* (swipe). Tap zones (left third — back, right two thirds —
   forward) work in both modes. The PDF reader keeps page-by-page
   navigation; the setting changes only swipe/tap direction, never the
   page-based position model (ADR-004). Position semantics are unchanged:
   the mode affects presentation only, `charOffset`/`page` stay the
   progress kinds.
3. **Library as a cover grid.** The Library screen renders an adaptive
   2–3 column grid of cover cards (cover/placeholder, title, author,
   progress percent). Sort order stays "recently read first".
4. **Welcome screen on first launch.** First run shows a welcome screen
   describing the two modes with actions "Sign in with Google" and
   "Skip — local mode"; skipping enters LOCAL mode (the library screen
   follows immediately). The screen is shown once (and after sign-out on
   next launch).
5. **Book deletion dialog.** Deleting a book asks the user:
   - **"Delete everywhere"** (Cloud mode, book with `driveFileId`):
     removes the Room row + local file, then the Drive book file, then the
     Drive progress JSON (`Nire-Liburu/progress/<bookId>.json`). Drive
     deletions are retried by WorkManager on failure; a 404 counts as
     success. Local removal happens first and is never deferred.
   - **"Delete locally only"** (Cloud mode, book with `driveFileId`):
     removes the local file; the row becomes metadata-only
     (`needsDownload = true`) and stays in the library; the Drive copy and
     progress JSON remain; the file re-downloads on next open.
   - Books without `driveFileId` (LOCAL mode, or local-only books in
     Cloud mode) get a single destructive action: delete row + local
     file; nothing exists in the cloud.

Supersedes in part:

- [ADR-006](./006-v1-scope-localization-distribution.md) — v1 now includes
  the fb2 TOC, the reader navigation-mode setting, the grid layout, the
  welcome screen, and the deletion dialog; themes/fonts/margins and the
  rest of the deferred list stay out of scope.

## Consequences

Positive:

- TOC costs little (the parser already extracts sections) and resolves the
  ADR-004/006 contradiction.
- Navigation-mode choice covers both common reader preferences without a
  position-model change.
- Grid + welcome screen fix the two most visible first-run UX gaps.
- Deletion semantics are explicit for every mode; "delete locally only"
  doubles as free-up-space.

Negative:

- The reader settings screen is a v1 surface ADR-006 deferred (only this
  one setting lives there in v1).
- "Delete locally only" keeps the card in the library; a user expecting
  full removal must pick "delete everywhere" (the dialog labels carry this
  burden).
- Grid layout needs placeholder covers for txt/pdf (no cover image).

## Alternatives Considered

- **Tap-only navigation (no scroll modes)** — rejected by owner decision:
  vertical scroll by default with an opt-in horizontal mode.
- **List layout** — rejected by owner decision: grid.
- **Immediate library on first launch (no welcome screen)** — rejected by
  owner decision: welcome screen.
- **Always-full deletion (no dialog)** — rejected by owner decision: the
  dialog distinguishes device-only vs account-wide deletion.
