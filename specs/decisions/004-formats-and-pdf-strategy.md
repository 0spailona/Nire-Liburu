# ADR-004: Supported formats and PDF rendering strategy

## Status

Accepted

## Context

The product must open plain text, FB2, and PDF books. Text formats (txt, fb2)
re-flow naturally to any screen and share a position model; PDF has fixed
layout where re-flow is a research-grade problem. The owner chose "native
PDF rendering first, reflow later as an improvement".

## Decision

Version 1 supports three formats:

| Format | Parsing | Rendering | Progress position |
| ------ | ------- | --------- | ----------------- |
| `.txt` | charset-detected plain text (UTF-8 default, fallback windows-1251 / UTF-16 via BOM and heuristic) | paginated re-flowed text | `charOffset` |
| `.fb2` | XML pull-parsing: title, author, chapters (`body/section`), paragraphs | same re-flowed text engine, chapter navigation | `charOffset` in the flattened text model |
| `.pdf` | Android `android.graphics.pdf.PdfRenderer` | native page-by-page bitmap rendering | `page` number |

- txt and fb2 share one pagination engine and one progress kind
  (`charOffset`), giving them identical UX.
- `.fb2.zip` archives (a common Russian-book distribution format) are
  accepted at import: the single inner `.fb2` is unpacked and takes the
  standard fb2 path. `bookId`, local storage, and Drive upload all use the
  unpacked inner `.fb2` bytes — a zipped and a plain copy of the same book
  share one identity.
- PDF progress is the 0-based page index; percent is `page / pageCount`.
- PDF reflow (text extraction into the shared engine) is explicitly deferred:
  it may arrive as a future ADR, not in v1.
- A format is detected by content signature first, file extension second.

## Consequences

Positive:

- One engine for two of three formats — less code, uniform txt/fb2 UX.
- `PdfRenderer` is a platform API: no third-party PDF dependency, no license
  burden, predictable behavior on API 26+.
- Deterministic progress kinds per format (see ADR-002 schema).

Negative:

- `PdfRenderer` renders one page at a time into a `Bitmap` — large pages on
  low-RAM devices require careful bitmap recycling and a page cache budget.
- Password-protected PDFs: `PdfRenderer` supports opening with a password in
  constructor; v1 surfaces a password prompt only if encountered, otherwise
  such files fail with a clear error.
- PDF text selection/search is out of scope in v1 (bitmap rendering only).
- Encrypted/corrupted FB2 must fail gracefully to raw-text fallback or an
  error, defined in the reading domain spec.
- `.fb2.zip` import needs temporary disk space for unpacking and one extra
  import branch (archive with ≠1 inner file fails with a clear error).

## Alternatives Considered

- **Third-party PDF libs (PdfiumAndroid, mupdf)** — rejected for v1: extra
  native dependencies and licenses; `PdfRenderer` covers the beta need.
- **PDF text extraction/reflow in v1** — rejected: high complexity, uncertain
  quality on real-world books; revisit after v1 as a separate ADR.
- **EPUB support** — out of scope for v1; a natural future extension via the
  same re-flow engine used by txt/fb2.
