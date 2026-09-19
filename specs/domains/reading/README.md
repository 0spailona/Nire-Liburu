# Reading Domain

## Purpose

Renders books on screen and produces/consumes reading positions: a paginated
re-flow engine for txt/fb2 (vertical scroll or horizontal paging, plus an
fb2 chapter sheet), and a page renderer for PDF.

## Key Files

- `app/src/main/kotlin/…/reading/ReaderViewModel.kt` — reader screen state,
  position persistence triggers <!-- TODO: verify when code exists -->
- `app/src/main/kotlin/…/reading/TextPaginators.kt` — txt/fb2 pagination
  (CharSequence → pages of char ranges)
- `app/src/main/kotlin/…/reading/Fb2Parser.kt` — XML pull parser →
  chapters + flattened paragraph text
- `app/src/main/kotlin/…/reading/ChapterSheet.kt` — flat fb2 TOC sheet,
  jump to chapter `charOffset` (ADR-008) <!-- TODO: verify when code
  exists -->
- `app/src/main/kotlin/…/reading/PdfPageRenderer.kt` — PdfRenderer wrapper
  with bitmap recycling and page cache
- `app/src/main/kotlin/…/reading/ReadingPosition.kt` — position model
- `app/src/main/kotlin/…/reading/ReaderMenuPanel.kt` — floating-button
  menu: position indicator, seek slider, actions (ADR-010)
  <!-- TODO: verify when code exists -->

## Core Types

```kotlin
sealed interface ReadingPosition {
    val percent: Float

    data class CharOffset(val charOffset: Int, val percent: Float) : ReadingPosition
    data class Page(val page: Int, val pageCount: Int) : ReadingPosition {
        override val percent: Float get() = page.toFloat() / pageCount
    }
}

data class BookContent(          // txt/fb2 shared engine input
    val fullText: String,        // flattened body text
    val chapters: List<Chapter>, // empty for txt
)

data class Chapter(              // flat TOC (ADR-008)
    val title: String,
    val charOffset: Int,         // first char of the chapter in fullText
)

enum class NavMode { VERTICAL_SCROLL, HORIZONTAL_PAGING } // reader setting

data class ReaderMenuState(      // reader menu panel (ADR-010)
    val visible: Boolean,
    val positionLabel: String,   // "62% · стр. 214 из 345" / "стр. 12 из 90"
    val percent: Float,          // seek slider value 0..1
)
```

## Flow

```
open book (library)
   ▼
Sync: pull progress (cloud mode) → LWW → effective position   [ADR-003]
   ▼
format switch ──┬─ TXT/FB2 ─► parse → paginate → render page at charOffset
                └─ PDF ─────► open PdfRenderer → render page N bitmap
   ▼
user reads; position changes in memory
   ▼
floating menu button (always on screen, corner — ADR-010)
   └─ tap ─► reader menu panel:
        ├─ title + position indicator "percent · page x of N"
        ├─ seek slider → charOffset (txt/fb2) / page (pdf);
        │   release persists position (same triggers as page turn)
        ├─ "chapters" (fb2 only — ADR-008/010; pdf: no outline in v1)
        ├─ "reader settings" → Settings screen Reader section (ADR-010)
        └─ "close book" → library (persist + enqueue push per ADR-003)
   ▼
txt/fb2: navigation per NavMode setting (ADR-008)
   ├─ VERTICAL_SCROLL (default): continuous scroll; tap zones jump
   │   backward/forward a screen
   └─ HORIZONTAL_PAGING: horizontal swipe between pages; tap zones work
pdf: page-by-page (swipe/tap); NavMode affects presentation only
     pinch-zoom + pan within page (ADR-009); zoom resets on page turn
   ▼
fb2: chapter sheet (flat list of Chapter titles) → jump to charOffset
   ▼
close book: Room UPDATE progress ─► Sync: enqueue push (cloud mode)
(process death: Room saved on each page turn / onStop)
```

## Invariants

- `ReadingPosition` is fully determined by book format: `CharOffset` for
  txt/fb2, `Page` for PDF (ADR-002/004).
- The navigation mode (`NavMode`) affects presentation only; it never
  changes the position kind or the persisted position semantics
  (ADR-008).
- The fb2 reader always offers a chapter sheet built from the flat
  `Chapter` list; selecting a chapter always lands on that chapter's
  `charOffset` (ADR-008).
- The reader menu floating button is always present while the reader is
  visible and never overlaps the tap zones (ADR-010).
- The seek slider maps onto the same position kinds (`charOffset`/`page`)
  and persists through the same triggers as page turns; it never creates
  a third position kind (ADR-010).
- PDF zoom is presentation-only: it never changes the `page` position
  kind, is never persisted, and resets to fit-width on every page turn
  (ADR-009).
- The app is locked to portrait in v1 (ADR-009); the reader never handles
  rotation-driven recreation or re-pagination.
- No full-text search and no text selection/copy in v1 (ADR-009); reader
  text is rendered non-selectable.
- The PDF reader offers no outline/chapter list in v1; the chapters
  action is fb2-only (ADR-010).
- Page layout depends only on (text content, viewport size, default text
  style in v1); identical inputs produce identical pagination.
- The reader functions with network disabled in both modes.
- PDF rendering recycles the previous page bitmap before rendering the next;
  at most 3 pages stay cached (~current ±1) <!-- TODO: verify budget -->.
- A position is persisted to Room at minimum on page turn, on close, and on
  `onStop`.

## Configuration

- Default reading font/style: single system default in v1 (no user themes —
  ADR-006).
- Reader navigation mode setting (ADR-008): `VERTICAL_SCROLL` (default) or
  `HORIZONTAL_PAGING`; lives in the Settings screen Reader section
  (ADR-010); tap zones (left third — back, right two thirds — forward)
  active in both modes and in the PDF reader.
- Reader menu (ADR-010): floating button in a screen corner,
  semi-transparent; panel shows position indicator (`percent · page x of
  N`; PDF: `page x of N`) and seek slider; actions: chapters (fb2),
  reader settings, close book.
- Reader window behavior (ADR-006): `FLAG_KEEP_SCREEN_ON` while the reader
  is visible; immersive fullscreen — status and navigation bars hidden,
  revealed by system gesture.
- Orientation: portrait lock for the whole app in v1 (ADR-009); landscape
  is a post-v1 extension point.
- PDF zoom (ADR-009): pinch-zoom + pan within the file page, fit-width
  default, reset on page turn; a tap is distinguished from a pinch/drag
  gesture so tap zones keep working while zoomed.
- No full-text search and no text selection/copy in v1 (ADR-009).
- Charset detection for txt: BOM → UTF-8 validity check → windows-1251
  heuristic fallback.

## Extension Points

- Reading customization (themes/fonts) extends the layout inputs; position
  model unaffected.
- Landscape orientation would unlock rotation handling in the reader
  (re-pagination on viewport change) — no position-model change (ADR-009).
- Full-text search would map matches onto `charOffset` — no position-model
  change (ADR-009).
- PDF outline (custom outline-table parser) would add a chapters action to
  the PDF reader menu — no position-model change; page targets map to
  `Page` (ADR-010).
- PDF reflow would add a third branch producing `BookContent` from PDF text
  extraction — new ADR required.
- TTS would consume `BookContent.fullText` with positions mapped back.

## Related Specs

- [System Overview](architecture/overview.md) — open/close flows
- [ADR-004](decisions/004-formats-and-pdf-strategy.md) — engine split
- [Library Domain](domains/library/README.md) — book file source
- [Sync Domain](domains/sync/README.md) — position pull/push triggers
