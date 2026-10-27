# 0001: Wails v2, not v3

- Date: 2026-10-25
- Status: accepted
- Decided in: M0 task 02 (prd/uwu/m0-spike)

## Context

uwu's shell is Wails (PLAN.md §3). The version rule agreed in the M0 grill: **v2 unless v3 has a stable (non-alpha/beta) release at spike time.**

Release status checked 2026-10-25 against the GitHub releases API (`wailsapp/wails`):

- Latest v3 tag: `v3.0.0-beta.34`, published 2026-10-20, marked prerelease. The entire v3 line is alpha/beta nightlies.
- Latest stable (non-prerelease) release: `v2.17.0`, published 2026-10-18.

v2 is still actively maintained (a stable release two days before the v3 beta above) and remains the project's documented stable line.

## Decision

Wails **v2**, pinned at spike time to `v2.17.0`. CLI installed via `go install github.com/wailsapp/wails/v2/cmd/wails@v2.17.0` (user-local `~/go/bin`, no system-wide changes).

## Consequences

- The spike and M2's shell build on v2 APIs: `wails init` v2 project layout, v2 bindings generation, v2 runtime events (`runtime.EventsEmit` / `EventsOn`).
- Wails-specific code stays confined to the app façade and the events bridge (PLAN.md §11 mitigation), so a later v2→v3 migration touches two seams, not the codebase.
- Revisit trigger: a stable v3.0.0 release before M2 starts would justify re-opening this record, not silently switching.
