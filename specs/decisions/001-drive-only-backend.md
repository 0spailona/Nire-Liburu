# ADR-001: Serverless architecture on Google Drive

## Status

Accepted (partially superseded by [ADR-007](./007-progress-in-visible-drive-folder.md):
progress storage location and OAuth scope list)

## Context

The app needs Google authentication, cloud book storage, and reading-progress
sync across devices, without running any own backend infrastructure. The owner
evaluated three families of options:

1. Firebase (Auth + Firestore + Storage)
2. Google Drive as storage + Firestore for progress
3. Own backend (Kotlin/Go + Postgres + S3)

During discussion the specific question "can reading progress be stored in
Google Drive?" was raised and answered affirmatively based on the official
Drive API documentation of the application data folder
(`developers.google.com/drive/api/guides/appdata`), which changed the tradeoff
landscape: a single-API, zero-cost, zero-infra solution became possible.

## Decision

Adopt **Drive-only** architecture with no backend of our own:

- Authentication — Google Identity (OAuth 2.0), requested scopes:
  `drive.appdata` (hidden progress data) and `drive.file`
  (per-file access to books the app created/opened itself).
- Books — files inside a visible folder named `Nire-Liburu` in the root of the
  user's My Drive. The user can manage the library from the Drive web UI.
- Reading progress — JSON files in the hidden `appDataFolder` space.
- Sync — pull on book open, push on book close, deferred retries via
  WorkManager. No realtime listeners, no push notifications.
- The app never runs nor requires any server component.

## Consequences

Positive:

- Zero infrastructure, zero hosting cost; nothing to monitor or back up.
- One API family (Google Drive API v3) for all cloud concerns.
- The 15 GB user quota covers both books and progress; progress JSON is tiny.
- `drive.appdata` and `drive.file` are non-sensitive scopes — lighter OAuth
  consent, no restricted-scope verification burden for beta distribution.
- Library is user-visible in Drive web UI; books survive app uninstall and
  can be managed manually.

Negative:

- No built-in offline write queue (Firestore has one): offline pushes are
  retried by our own WorkManager policy.
- Conflict resolution is client-side last-write-wins (see ADR-002); Drive
  offers no realtime transactions.
- Drive API rate limits (~1000 req/min) apply; acceptable because sync happens
  only on book open/close.
- Vendor lock-in to Google Drive as the storage model.

## Alternatives Considered

- **Firebase (Auth + Firestore + Storage)** — rejected: introduces a second
  ecosystem beyond Google Identity/Drive, Storage quotas and billing rules for
  a personal beta, while Firestore's offline queue is the only major gain
  (replaceable by WorkManager retries under the open/close sync model).
- **Google Drive for books + Firestore for progress** — rejected: two APIs to
  maintain, two consistency domains, more client code, with no user-visible
  benefit over Drive-only.
- **Own backend (Kotlin/Go + Postgres + S3)** — rejected: full control but
  weeks of extra work, server/domain/TLS/backups burden, manual Google OAuth
  token verification; clear overkill for a personal beta.
