# ADR-011: Library grid interactions — never-read sort rule, delete entry point

## Status

Accepted

## Context

Two last UX details were left unspecified after ADR-010:

1. ADR-008 fixed the grid sort as "recently read first", but the placement
   of never-read books (null `lastReadAt`) carried a
   `<!-- TODO: verify UX decision -->` marker in the library domain spec.
2. The deletion dialog (ADR-008) had no described entry point. The grid
   card is fully consumed by tap-to-open, the reader is immersive
   fullscreen with tap zones (ADR-010), and no book-details screen exists
   in v1 — so no surface for "delete" had been defined.

## Decision

1. **Never-read books sort last.** Confirmed: the grid sorts by
   `lastReadAt` descending; books with null `lastReadAt` always come after
   all read books. The `TODO: verify UX decision` marker is removed.
   The order inside the never-read group is an implementation detail and
   stays unspecified in v1 (<!-- TODO: verify --> when code lands).
2. **Delete entry point: long-press on a grid card.** Long-press on a
   library card opens the ADR-008 confirmation dialog ("delete everywhere"
   / "delete locally only", or the single destructive action for books
   without `driveFileId`). A plain tap always opens the book; there is no
   per-card delete control and no batch selection in v1.

Supersedes in part:

- [ADR-008](./008-v1-scope-revision.md) — adds the grid interaction layer
  (confirmed never-read placement, long-press entry point) on top of the
  dialog semantics and sort order defined there; nothing in ADR-008 is
  overturned.

## Consequences

Positive:

- The grid stays clean — no per-card chrome; a destructive action is hard
  to trigger accidentally (long-press plus the ADR-008 confirmation).
- The sort order is fully deterministic; the last open UX question in the
  spec system is closed.

Negative:

- Long-press is not discoverable; the welcome/empty-state copy and the
  Settings screen must mention it (first-sprint copy task).
- Deleting many books is a one-by-one operation until batch selection
  (post-v1).

## Alternatives Considered

- **Trash icon on every card** — rejected by owner decision: clutters the
  grid and puts a destructive action one tap away.
- **Delete via a book-details screen** — rejected by owner decision: v1
  has no book-details screen, and adding one for a single action is not
  worth it.
- **Never-read books first** — rejected by owner decision: recently read
  on top, never-read at the end.
