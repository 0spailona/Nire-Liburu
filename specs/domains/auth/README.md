# Authentication Domain

## Purpose

Handles optional Google sign-in via Credential Manager, issues and refreshes
the OAuth token consumed by the Drive client, and exposes the current
operating mode (Local / Cloud) to the rest of the app.

## Key Files

- `app/src/main/kotlin/…/auth/SignInViewModel.kt` — screen state and sign-in
  actions <!-- TODO: verify when code exists -->
- `app/src/main/kotlin/…/auth/WelcomeScreen.kt` — first-launch welcome
  screen: "Sign in with Google" / "Skip — local mode" (ADR-008)
  <!-- TODO: verify when code exists -->
- `app/src/main/kotlin/…/auth/SettingsScreen.kt` — Settings screen: Account
  section (email, mode, sign-out) + Reader section (NavMode); reachable
  from the Library toolbar (ADR-010) <!-- TODO: verify when code exists -->
- `app/src/main/kotlin/…/auth/AccountRepository.kt` — wraps Credential
  Manager, stores the active Google account <!-- TODO: verify -->
- `app/src/main/kotlin/…/auth/TokenProvider.kt` — returns a valid access
  token for Drive scopes, performing silent refresh <!-- TODO: verify -->

## Core Types

```kotlin
enum class OperatingMode { LOCAL, CLOUD }

data class AccountState(
    val mode: OperatingMode,
    val email: String?,      // null in LOCAL mode
    val displayName: String?,
)
```

## Flow

```
first launch ──► WELCOME screen (shown once — ADR-008)
      │            actions: "Sign in with Google" · "Skip — local mode"
      ├─ skip ────────────────► LOCAL mode (no account) → Library screen
      └─ "Sign in with Google" ──► Credential Manager sheet
                                       │ GetGoogleIdOption
                                       │ scopes: drive.file (ADR-007)
                                       ▼
                              GoogleIdToken ──► AccountRepository saves account
                                       │
                        same account as before? ──yes──► CLOUD (no dialog)
                                       │ no / local books present
                                       ▼
                     choice dialog (ADR-003):
                     "upload local books" ──► one-time merge (sync domain)
                     "use this account's library" ──► clear driveFileId
                       on local books, pull new account's books (library
                       refresh); local books become local-only

Settings screen (ADR-010; entry from Library toolbar):
   ├─ Account section: email + operating mode + "sign out"
   │    sign-out ──► mode = LOCAL (local books & progress stay intact;
   │                 driveFileId values are kept for same-account
   │                 re-sign-in; welcome screen shown again on next
   │                 launch — ADR-008)
   └─ Reader section: navigation mode (NavMode, ADR-008; default
      vertical scroll)
```

## Invariants

- The app starts in LOCAL mode and remains fully functional without an
  account.
- The welcome screen appears exactly once per LOCAL-mode session start:
  on first launch and on the first launch after a sign-out; it never
  appears while an account is active (ADR-008).
- Sign-out lives only on the Settings screen Account section; it is the
  single sign-out surface in the app (ADR-010).
- A valid access token is obtained lazily and refreshed silently before any
  Drive call.
- Sign-out deletes locally cached tokens and never deletes books or progress.
- Exactly zero or one account is active at any time.
- Sign-in to a different account always shows the choice dialog (ADR-003);
  re-sign-in to the same account never does.

## Configuration

- OAuth scope: `drive.file` (non-sensitive; the only scope — ADR-007;
  `drive.appdata` is not requested).
- OAuth client id: per build type, in `local.properties` / CI secrets
  (never committed) <!-- TODO: verify mechanism during setup -->.
- UI language: `values/` = Russian (default), `values-en/` = English;
  any other device locale falls back to Russian; no in-app language
  switcher in v1 (ADR-010 confirms ADR-006).

## Extension Points

- Adding "sign in with passkeys/other providers" extends `AccountRepository`
  without touching mode logic.
- A server backend, if ever added (ADR-001 alternative), would replace
  `TokenProvider` only.
- An in-app language switcher (per-app languages) would extend the
  Settings screen without touching mode or token logic (ADR-010).

## Related Specs

- [System Overview](architecture/overview.md) — layering and flows
- [ADR-001](decisions/001-drive-only-backend.md) — scopes, serverless
- [ADR-003](decisions/003-dual-mode-open-close-sync.md) — mode semantics
- [Drive Storage Contract](contracts/drive-storage.md) — token usage
