# Spec Index

Navigation for this project's specification system. Find your task, read the listed specs.

> **Design-stage specs:** these specs describe an agreed design for which
> code does not exist yet (bootstrap phase). Paths in `Key Files` sections
> are planned locations and carry `<!-- TODO: verify -->` markers. When the
> first code lands, verify each claim against the actual code and remove
> this notice.

## Task → Spec Table

| Task | Read |
| ---- | ---- |
| Understand spec formats / add a new spec | [META.md](META.md) |
| Learn how to work with the spec system | [WORKFLOW.md](WORKFLOW.md) |
| Plan implementation work / find what to build next | [Implementation Roadmap](roadmap.md) |
| Get the big picture / onboard | [System Overview](architecture/overview.md) |
| Understand system layering and invariants | [System Overview](architecture/overview.md) |
| Work on Google sign-in / operating modes | [Auth Domain](domains/auth/README.md) |
| Import, metadata, library list | [Library Domain](domains/library/README.md) |
| Readers (txt/fb2 engine, PDF renderer), positions | [Reading Domain](domains/reading/README.md) |
| Progress sync, retries, mode upgrade | [Sync Domain](domains/sync/README.md) |
| Change Drive layout / progress schema / add API calls | [Drive Storage Contract](contracts/drive-storage.md) |
| Record an architectural decision | [decisions/_template.md](decisions/_template.md) |
| Why serverless Drive-only | [ADR-001](decisions/001-drive-only-backend.md) |
| Why content-hash bookId / LWW / progress JSON | [ADR-002](decisions/002-book-identity-and-progress-model.md) |
| Why open/close sync points / dual modes | [ADR-003](decisions/003-dual-mode-open-close-sync.md) |
| Why PdfRenderer / shared txt+fb2 engine | [ADR-004](decisions/004-formats-and-pdf-strategy.md) |
| Stack choices (Kotlin, Compose, Room, …) | [ADR-005](decisions/005-technology-stack.md) |
| What is in/out of v1 | [ADR-006](decisions/006-v1-scope-localization-distribution.md) |
| Why progress lives in the visible folder / single scope | [ADR-007](decisions/007-progress-in-visible-drive-folder.md) |
| Why TOC, nav modes, grid, welcome, deletion dialog | [ADR-008](decisions/008-v1-scope-revision.md) |
| Why portrait lock, PDF zoom, backup opt-out / no search, rename, selection | [ADR-009](decisions/009-reader-interaction-and-backup.md) |
| Why floating menu button, settings screen, RU default, multi-import, 20 MB guard, seek | [ADR-010](decisions/010-reader-chrome-settings-language-ux.md) |
| Why never-read sort last / delete via card long-press | [ADR-011](decisions/011-library-grid-sort-and-delete-entry.md) |

## Dependency Graph

```
architecture/overview.md ──┬──► domains/auth/README.md ──────► contracts/drive-storage.md
                           │                                        ▲
                           ├──► domains/library/README.md ──────────┤
                           │                                        │
                           ├──► domains/reading/README.md ──► domains/sync/README.md
                           │
                           └──► decisions/001…011 (rationale for all of the above)
```

Reading order for a new contributor: overview → ADRs 001–003, 007 → domain
of interest → contract.

## Directory Listing

```
specs/
├── META.md          format rules, templates, update protocol
├── INDEX.md         this file
├── WORKFLOW.md      how to work with the spec system
├── roadmap.md       implementation roadmap (phases, tasks, acceptance criteria)
├── architecture/
│   └── overview.md  system map, layers, flows, invariants
├── domains/
│   ├── auth/README.md      sign-in, modes, token provider
│   ├── library/README.md   import, metadata, catalog
│   ├── reading/README.md   txt/fb2 engine, PDF renderer, positions
│   └── sync/README.md      progress sync, retries, mode upgrade
├── contracts/
│   └── drive-storage.md    app <-> Google Drive boundary
└── decisions/
    ├── _template.md
    ├── 001-drive-only-backend.md
    ├── 002-book-identity-and-progress-model.md
    ├── 003-dual-mode-open-close-sync.md
    ├── 004-formats-and-pdf-strategy.md
    ├── 005-technology-stack.md
    ├── 006-v1-scope-localization-distribution.md
    ├── 007-progress-in-visible-drive-folder.md
    ├── 008-v1-scope-revision.md
    ├── 009-reader-interaction-and-backup.md
    ├── 010-reader-chrome-settings-language-ux.md
    └── 011-library-grid-sort-and-delete-entry.md
```
