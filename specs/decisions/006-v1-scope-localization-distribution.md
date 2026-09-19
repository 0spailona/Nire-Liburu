# ADR-006: Scope of version 1, localization, and distribution

## Status

Accepted (partially superseded by [ADR-008](./008-v1-scope-revision.md):
v1 additionally includes the fb2 TOC, reader navigation-mode setting,
grid layout, welcome screen, and deletion dialog; partially superseded by
[ADR-009](./009-reader-interaction-and-backup.md): v1 additionally
includes the PDF pinch-zoom and the portrait lock; the out-of-scope list
additionally names full-text search, book renaming, and text
selection/copy; partially superseded by
[ADR-010](./010-reader-chrome-settings-language-ux.md): v1 additionally
includes the settings screen, multi-select import, the metered-download
guard, and the reader position indicator/seek; the Russian-default
language fallback is confirmed)

## Context

The owner defined the target audience (open beta via APK link, without Google
Play), UI languages (Russian + English), the book source (files from the
device via SAF), and explicitly chose the minimal feature set for v1
("only base reading of the three formats + progress sync").

## Decision

**In scope for v1:**

- Library screen: list of imported books with title/author/cover (fb2),
  sort by recent progress.
- Import books from device storage via Storage Access Framework (SAF) file
  picker (txt, fb2, fb2.zip, pdf). Re-importing a book whose content hash
  already exists in the library is a no-op with a notice "already in
  library" — reading position untouched (owner's choice).
- Reading screens: re-flowed text reader (txt/fb2) and page reader (PDF).
- Reader screen behavior: keep screen on (`FLAG_KEEP_SCREEN_ON`) while
  reading and immersive fullscreen (status/navigation bars hidden, revealed
  by system gesture) — owner's explicit choice for v1.
- Reading progress: local persistence always; cloud sync per ADR-003 when
  signed in.
- Google sign-in (optional), account state visible in UI, sign-out.
- Local ⇄ Cloud mode upgrade with one-time merge.
- New-device restore: metadata-only library rows from Drive, file downloads
  on first open.
- Light theme only (single light Material 3 scheme); the owner explicitly
  deferred dark theme and any reading customization past v1.
- UI languages: Russian and English (`values/`, `values-en/`).
- Distribution: unsigned-by-Play open beta; APK shared directly (e.g. via
  GitHub Releases), self-signed.

**Explicitly out of scope for v1** (candidates for later versions):

- Bookmarks, highlights, notes
- Reading customization: themes, fonts, margins, brightness
- Dark theme (including system dark-mode following)
- TTS / narration
- Collections, folders, tags
- PDF reflow, PDF text search/selection
- OPDS catalogs, online bookstores
- iOS/desktop clients

## Consequences

Positive:

- Sharp v1 boundary keeps the beta small, testable, and shippable.
- APK-based beta avoids Play review, signing, and policy overhead; the two
  non-sensitive Drive scopes keep OAuth consent light.
- SAF import covers the "files from device" requirement with zero storage
  permissions (no READ_EXTERNAL_STORAGE / MANAGE_STORAGE).

Negative:

- Direct APK distribution means manual update delivery and no crash
  reporting from Play; consider Firebase Crashlytics later (rejected in v1 to
  keep zero third-party platform deps).
- Only two locales; string resources must be designed for addition of more.
- Later additions (bookmarks, themes) will require progress-format
  extensions — the `schema` field in the progress JSON (ADR-002) reserves
  that path.

## Alternatives Considered

- **Google Play open/closed beta** — deferred: privacy policy and listing
  requirements are overhead for a friends-and-family beta; can be added
  later without architectural change.
- **Wider v1 feature set (bookmarks, themes, TTS)** — rejected by owner
  decision for v1; revisit post-beta.
