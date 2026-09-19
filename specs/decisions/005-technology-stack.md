# ADR-005: Technology stack

## Status

Accepted (partially superseded by [ADR-007](./007-progress-in-visible-drive-folder.md):
OAuth scope note in the Drive row)

## Context

A greenfield Android app (bootstrap repository, no code yet). The stack must
be modern, long-term maintainable by a single developer, and fit the Drive-only
architecture (ADR-001) and the open/close sync model (ADR-003).

## Decision

| Concern | Choice | Notes |
| ------- | ------ | ----- |
| Language | Kotlin (JVM) | single language everywhere |
| Min SDK | 26 (Android 8.0) | covers ~97% of active devices |
| Target SDK | latest stable at build time | required for beta APK distribution |
| UI | Jetpack Compose + Material 3 | declarative UI, single-activity |
| Architecture pattern | MVVM + unidirectional data flow (UDF) | ViewModel + StateFlow |
| DI | Hilt | standard Compose-era DI |
| Async | Coroutines + Flow | incl. Drive API calls |
| Local DB | Room | books metadata, progress cache, pending sync queue |
| Google sign-in | Credential Manager + GoogleIdToken / Google Identity API | modern replacement for the deprecated GoogleSignIn SDK |
| Drive | Google Drive API v3 (REST, via generated client or OkHttp/Retrofit wrapper) | scopes `drive.appdata` + `drive.file` |
| Background work | WorkManager | retry of failed progress pushes |
| Serialization | kotlinx.serialization | progress JSON (ADR-002) and settings |
| Images (FB2 covers) | Coil | cover thumbnails in library |
| Build | Gradle (Kotlin DSL), version catalog | single module at first; split if needed |
| Tests | JUnit 5 + MockK + Turbine (Flows) + Robolectric for Android types | unit-first; UI tests later |

Application identity: `applicationId` = `com.nireliburu` (chosen once,
permanent for the APK's lifetime; namespace matches).

## Consequences

Positive:

- All-Jetpack-modern stack; no legacy View system except where platform APIs
  demand (PdfRenderer interop is thin).
- Room as the single local source of truth makes offline mode and the pending
  push queue straightforward.
- Credential Manager is the currently recommended Google auth path; the legacy
  GoogleSignIn API is deprecated.

Negative:

- Credential Manager + Drive REST is a slightly less trodden path than
  Firebase Auth; OAuth token refresh for Drive must be handled manually
  (Google Auth library / `GoogleAuthorizationCodeToken` flow, or
  `CredentialManager`-issued token with silent refresh).
- Drive API v3 has no first-class Kotlin coroutine client; a thin Retrofit
  wrapper or Google's `google-api-services-drive` with coroutine adapters is
  required <!-- TODO: verify exact client choice during implementation -->.

## Alternatives Considered

- **XML Views + Material Components** — rejected: legacy paradigm for a new
  app.
- **KMP/multiplatform** — rejected: no iOS target, extra constraints.
- **SQLDelight instead of Room** — viable, but Room's Compose-era tooling and
  coroutines support fit better.
- **RxJava** — rejected: superseded by coroutines/Flow for greenfield.
