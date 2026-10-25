---
type: roadmap
repo: uwu
status: in-progress
created: 2026-10-25
updated: 2026-10-25
tags:
  - roadmap
  - step-by-step
---

# uwu Roadmap

Provisional PRDs for M1–M7: goal, dependencies, and open questions only. Per the agile decision (M0 grill, Q1), these stay short because M0's outcomes are expected to change them. A milestone **graduates** when its turn comes: it gets its own task folder under `prd/uwu/` with a full PRD, a task table, and walkthroughs, and its entry here gains a link and a `graduated` marker.

M0 is graduated: [[uwu/m0-spike/prd-m0-spike|M0: Spike]].

Source of truth for architecture and milestone detail is `PLAN.md` in the repo root. This note tracks scope and open questions per milestone, not design.

## M1: Domain and store

**Goal:** domain types, state machine, invariants, SQLite repos, migrations, event log, markdown export, importer for existing `prd/` folders. Starts fresh — the M0 spike is deleted; M0's decision records and session fixtures are inputs.

**Depends on:** M0 (CLI vs SDK decision shapes the `AgentDriver` boundary; fixtures model the fake driver).

> [!question] Does the fake `AgentDriver` live in M1 or wait for M3?
> M1's engine tests need scripted sessions. The fixtures exist after M0; the replay harness has to live somewhere.

> [!question] Is `.uwu/` committed or git-ignored by default?
> PLAN.md open decision 2.

## M2: Read-only graph UI

**Goal:** Wails shell, React shell, Query and Zustand wiring, graph with wave layout (React Flow + ELK), task panel, live updates from events.

**Depends on:** M0 (Wails version, event-bridge pattern proven), M1 (domain and event log to render).

> [!question] Can Playwright drive `wails dev`'s browser endpoint?
> Carried from M0 PRD Q1 if unresolved there. Determines the frontend testing story.

## M3: Planning flow

**Goal:** grill, PRD, split approval, walkthrough generation in the UI, driven by `AgentDriver`.

**Depends on:** M1 (domain), M2 (UI surfaces), M0 (driver approach).

> [!question] How does the grill's one-question-at-a-time flow map onto an async agent session?
> The skill blocks on the user; an app can let the user walk away mid-grill.

## M4: Stage loop

**Goal:** scaffold, stub, red tests, write guard, brief gate, close stage. Test adapters: generic command, `go test -json`, vitest JSON, JUnit XML. **The core product.**

**Depends on:** M1–M3; M0's hook verdict and Bash-hole findings.

> [!question] Bash-hole strategy: detect-and-revert backstop, OS-level sandboxing, or both?
> Decided here, with M0's probe data (M0 PRD, task 09).

> [!question] Stub lock: how is "code that runs" detected for the static check?
> PLAN.md §8. Language-dependent; may start Go/TS-only.

## M5: Hints and review

**Goal:** on-demand graduated hints from the user's current diff, review view, commit suggestion.

**Depends on:** M4 (stage loop produces the diffs and the review moment).

> [!question] Hint generation: same agent session as the task, or a fresh one with the walkthrough as context?
> Cost, context quality, and leak risk (a hint must not reveal more than its level).

## M6: Parallel agent branches

**Goal:** agent works unblocked agent-owned nodes in git worktrees while the human works, merge on completion, conflict surfacing. The differentiator, and the riskiest piece — stays late deliberately.

**Depends on:** M4 (guard and stage loop), M1 (graph invariants).

> [!question] Merge policy when an agent branch and the human's working tree touch the same generated file?
> PLAN.md §11 says conflicts become a task, not a silent merge; the task shape is undesigned.

## M7: Polish

**Goal:** multi-project list, per-task level adaptation, fast path for small tasks, settings presets, packaging and signing for macOS.

**Depends on:** M1–M6.

> [!question] Where does human review happen under ownership `none`?
> PLAN.md open decision 5, deferred from M0 (2026-10-25): the current focus is the human-in-the-loop flow. Proposal on file: a final review gate before anything is called done; alternative: per-node review. Revisit when agent-autonomous flow becomes a priority — could land earlier than M7 if priorities shift.

## Change Log

- 2026-10-25: Created with M1–M7 provisional entries, per M0 grill decisions Q1 (provisional PRDs) and Q5 (single roadmap note, milestones graduate to folders).
