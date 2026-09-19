# ADR-010: Reader chrome, settings screen, language default, import & download UX

## Status

Accepted

## Context

Seven product decisions were made by the owner after ADR-009. An end-to-end
walk of the user path exposed surfaces the specs had not fixed: (1) the
reader is immersive fullscreen with tap zones covering the whole screen,
leaving no described way to open the TOC, settings, or leave the reader;
(2) the reader navigation-mode setting and sign-out had no home (a settings
screen was never designed); (3) ADR-006 set `values/` to Russian, so any
device locale other than ru/en falls back to Russian; (4) the import flow
was specified for a single file while the SAF picker supports
multi-selection; (5) first-open download of a large book could consume
100+ MB of metered data silently; (6) the reader showed no position
indicator and no seek affordance; (7) the PDF embedded outline (bookmarks)
had never been scoped.

## Decision

1. **Reader menu via floating button.** A small semi-transparent floating
   action button is always present in the reader corner. Tapping it opens
   the reader menu panel: book title, position indicator
   (`percent · page x of N`; PDF: `page x of N`), a seek slider mapped to
   `charOffset`/`page`, and actions: chapters (fb2), reader settings,
   close book (back to library). Tap zones (left third — back, right two
   thirds — forward) are unchanged and never overlap the button.
2. **Settings screen.** A dedicated Settings screen reachable from the
   Library toolbar, with two sections: *Account* (active account email
   and mode, sign-out) and *Reader* (navigation mode). It is the single
   home of the NavMode setting (ADR-008) and the only sign-out surface.
3. **Language default stays Russian.** `values/` = Russian,
   `values-en/` = English, unchanged from ADR-006: the owner's audience is
   Russian-speaking, so a non-ru/en device locale falls back to Russian.
   No in-app language switcher in v1.
4. **Multi-select SAF import.** The picker allows multiple files; each is
   imported sequentially through the single-file pipeline; duplicates are
   skipped with the "already in library" notice per file.
5. **Metered-download guard.** Before a first-open download on a metered
   network, the app checks the Drive file size: ≤ 20 MB downloads
   silently; > 20 MB shows a confirmation dialog ("book is X MB —
   download over mobile data?"). Wi-Fi never asks. Decline leaves the
   book as a metadata-only row; the next open re-prompts.
6. **Position indicator and seek slider in the reader menu.** The reader
   menu panel (item 1) shows `percent · page x of N` and a seek slider;
   dragging it moves `charOffset` (txt/fb2) or `page` (PDF) immediately;
   releasing persists the position (same triggers as page turn: Room
   write, push enqueued on close per ADR-003).
7. **PDF outline (bookmarks) stays out of v1.** `PdfRenderer` exposes no
   outline API; a custom PDF outline-table parser is deferred post-v1.
   The chapters action in the reader menu is fb2-only in v1.

Supersedes in part:

- [ADR-006](./006-v1-scope-localization-distribution.md) — v1
  additionally includes the settings screen, multi-select import, the
  metered-download guard, and the reader position indicator/seek; the
  language fallback (values/ = Russian) is confirmed rather than changed.

## Consequences

Positive:

- The floating button keeps tap zones intact and gives every reader
  action a single, discoverable home; the panel doubles as the position
  indicator surface.
- One settings screen centralizes NavMode and sign-out; the reader needs
  no inline settings UI.
- Multi-select import matches how users actually batch-add books.
- The 20 MB guard prevents silent metered-data surprises; the threshold
  is a constant easy to tune.
- The seek slider makes long (megabyte-scale) txt files navigable in
  vertical scroll mode.

Negative:

- The floating button covers a corner of the page; layouts must keep it
  from overlapping text meaningfully (semi-transparent, small).
- A user who declines a >20 MB download cannot read the book until
  re-prompted on next open (accepted friction).
- Non-ru/en locales see Russian (owner-accepted fallback).
- PDF readers get no chapter list in v1.

## Alternatives Considered

- **Center-tap toolbar toggle** — rejected by owner decision: floating
  button chosen instead.
- **Settings as reader-inline sheet only** — rejected by owner decision:
  dedicated screen from the Library toolbar.
- **English default (values/ = en)** — rejected by owner decision:
  Russian-speaking audience; Russian default kept.
- **In-app language switcher (per-app languages)** — rejected for v1:
  extra surface; system-locale resolution suffices.
- **Wi-Fi-only downloads for large books** — rejected by owner decision:
  confirm-dialog guard instead of hard blocking.
- **PDF outline in v1 via custom parser** — rejected: `PdfRenderer` has
  no outline API; a hand-rolled parser adds risk for marginal gain.
- **No seek slider (percent label only)** — rejected by owner decision:
  slider included.
