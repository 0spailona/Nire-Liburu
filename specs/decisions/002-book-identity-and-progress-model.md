# ADR-002: Book identity and progress model on Drive

## Status

Accepted (partially superseded by [ADR-007](./007-progress-in-visible-drive-folder.md):
progress file storage location)

## Context

Progress must sync across devices via Google Drive files only (ADR-001), and
the same book file may live on several devices with different local paths.
A stable, device-independent book identity is required, plus a progress format
that two devices can compare to resolve conflicts without a server.

## Decision

1. **Book identity** — content hash SHA-256 of the book file bytes, truncated
   to the first 16 hex chars (64 bits) as `bookId`. Identity is path- and
   device-independent: the same file uploaded from any device resolves to the
   same `bookId`.
2. **Local→cloud book linkage** — the Drive `fileId` of the uploaded book is
   stored in the local database per device; `bookId` is the join key for
   progress.
3. **Progress storage on Drive** — one JSON file per book in `appDataFolder`,
   named `progress/<bookId>.json`.
4. **Conflict resolution** — last-write-wins (LWW) by comparing an
   RFC 3339 UTC `updatedAt` timestamp (device clock) plus a monotonically
   increasing `revision` counter per book (incremented by the writing device);
   on equal timestamps the higher `revision` wins, on full tie the lexicically
   greater `deviceId` wins, making resolution deterministic and total.

Progress JSON schema (v1):

```json
{
  "schema": 1,
  "bookId": "0f2a9c…",
  "format": "fb2",
  "position": {
    "kind": "charOffset",
    "charOffset": 104857
  },
  "percent": 0.42,
  "deviceId": "a1b2c3d4",
  "revision": 17,
  "updatedAt": "2026-09-18T10:00:00Z"
}
```

For PDF, `position.kind` is `"page"` with `"page": 12` instead of
`charOffset`.

## Consequences

Positive:

- The same book added independently on two devices converges to one progress
  record automatically.
- Progress files are tiny, human-readable, debuggable via `files.list` with
  `spaces=appDataFolder`.
- Deterministic, total conflict ordering prevents flip-flopping progress.

Negative:

- Hashing large files costs time on import (mitigated: hash once at import,
  streaming; 64-bit truncation makes collisions negligible for a library of
  personal scale).
- LWW can lose a small amount of progress when two offline devices both
  advance the same book; accepted for a personal reader.
- Device-clock skew affects timestamps; the `revision` counter is the
  systematic tie-breaker, limiting skew impact.

## Alternatives Considered

- **Drive `fileId` as book identity** — rejected: the same content uploaded
  from two devices yields two `fileId`s, splitting progress history; also
  breaks local-mode books that were never uploaded.
- **One combined progress file for all books** — rejected: full-file rewrite
  on every book close multiplies conflict surface and payload size.
- **Per-position vector clocks** — rejected: overkill without a server to
  exchange clock state; LWW + revision covers the personal multi-device case.
