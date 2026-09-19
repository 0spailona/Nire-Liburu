# ADR-003: Dual operating modes and open/close sync points

## Status

Accepted (partially superseded by [ADR-007](./007-progress-in-visible-drive-folder.md):
progress file location in pull/push bullets)

## Context

The owner specified exactly two desired behaviors: (1) everything local —
books and progress; (2) books stored locally and in the cloud, with progress
synced with the backend on book open and close. A fully cloud-only streaming
mode was considered and rejected as needlessly fragile offline.

## Decision

The app supports two operating modes, explicit for the user:

1. **Local mode** — the app is fully functional without any Google sign-in:
   books and progress live only on the device. Signing in later upgrades to
   Cloud mode.
2. **Cloud mode** — active after Google sign-in: books are stored both locally
   and on Google Drive; reading progress syncs with the cloud exactly twice
   per reading session:
   - **Pull** — on book open: the freshest progress is fetched from
     `appDataFolder` before the reader UI is shown.
   - **Push** — on book close: the updated progress is written to
     `appDataFolder` when the user leaves the reader screen (and on app
     process death via lifecycle-safe persistence).

Sync triggers are deliberately limited to open/close: no background progress
sync, no periodic sync, no realtime listeners. The one foreground exception
is library *metadata* refresh: in Cloud mode `listBooks()` runs when the
Library screen is entered and on manual pull-to-refresh (the owner chose this
trigger over app-start-only); it never syncs progress and never runs in the
background.

Retry policy for failed pushes: WorkManager exponential backoff
(network-constrained), attempt limit 10; the local database remains the
source of truth until the push succeeds. Pull failures fall back to local
progress with a non-blocking notice.

Mode upgrade (local → cloud) performs a one-time merge: local books are
uploaded to Drive and local progress seeds the cloud records via the same
LWW rules (ADR-002).

Signing in to a **different** account than the previously active one (or the
first sign-in after a sign-out with local books present) shows a choice
dialog, per the owner's decision:

- **"Upload local books to this account"** — the standard mode-upgrade merge
  (upload books, seed progress via LWW).
- **"Use this account's library"** — local books stay on the device and stay
  visible in the library, but they become local-only: their `driveFileId`
  values are cleared (they point into the previous account's Drive and are
  meaningless for the new one), so their progress does not sync until the
  book is uploaded to the new account. The library then pulls the new
  account's Drive books via `listBooks()`.

Re-sign-in to the *same* account skips the dialog and resumes Cloud mode
directly.

New-device restore (cloud mode on a device with an empty local library):
`listBooks()` populates the library with metadata-only rows immediately;
book files download on first open (on-demand), verifying the content hash
on arrival. The owner explicitly chose on-demand over download-everything
to save bandwidth.

## Consequences

Positive:

- The app is usable without any account — zero-friction first run.
- Two sync points per session keep Drive API usage minimal and predictable;
  rate limits are a non-issue.
- Battery and privacy friendly: no background data flows.

Negative:

- Progress made on device A does not appear on device B until B opens the
  book (pull-on-open) — acceptable per requirements.
- An offline push queue can contain at most one pending progress write per
  book; ordering is guaranteed by the revision counter.

## Alternatives Considered

- **Cloud-only streaming reading** — rejected: reading requires network,
  worse UX offline, more Drive bandwidth.
- **Continuous/periodic background sync** — rejected: contradicts the
  open/close model agreed with the owner, wastes battery and quota.
- **Forced sign-in on first launch** — rejected: the local mode is a stated
  requirement.
