# uwu: Plan

Status: draft v2 (2026-10-23). Supersedes the earlier web-app sketch.

## 1. What uwu is

A desktop app for human-in-the-loop coding. A task graph is the main surface. Each node is owned either by you or by the agent. The agent works the nodes it owns while you work yours, and the app enforces who touches what.

- **From the `step-by-step` skill:** the seven-phase flow (grill, PRD, split approval, walkthroughs, scaffold, brief and stop, close stage), the ownership split, red tests that define done, hints, resumability.
- **From omo's UX (homepage demo only, not the product):** a graph laid out in waves, a status line per node, parallel threads, live progress.
- **New:** a human-owned node inside an agent graph, with enforcement instead of instruction.

The human share is configurable all the way down to zero (ownership `none`, section 6), so the same graph and review UI serve a fully agent-run job.

Not a goal: building an agent engine, matching omo's multi-agent swarms, or multi-model orchestration. The agent is a driver behind an interface.

## 2. Principles

1. **Enforce, don't ask.** The skill can only request "stop after the stage" or "don't touch my task". uwu blocks it.
2. **One source of truth.** State lives in SQLite. Markdown (PRD, walkthroughs, Obsidian-compatible) is an export, never an input. No drift checks, no filename-encoded dependencies.
3. **Guidance is a dial.** How much hand-holding you get is configurable per app, per project and per task (section 6).
4. **The agent is replaceable.** A narrow `AgentDriver` interface. Claude Code is the first implementation.
5. **Tests are the feedback loop.** You can run them yourself at any time, with or without the agent.

## 3. Stack

| Layer | Choice | Notes |
|---|---|---|
| Shell | Wails (Go + system webview) | v2 vs v3 decided in M0, based on current stability. Default is v2 if v3 is not released. |
| Backend | Go | State, agent driver, guard, test runner, git, export. Runs in-process in the Wails app. No separate server. |
| Persistence | SQLite via `modernc.org/sqlite` | Pure Go, no cgo, simpler cross-compiles. Stored in the target repo (`.uwu/`), so it travels with the project. |
| Frontend | React + TypeScript (strict), Vite | |
| Client state | Zustand | Ephemeral UI only: selection, open panels, drafts, streaming buffers. |
| Server state | TanStack Query | Anything that lives in Go. Wails bindings are the query functions. |
| Other TanStack | Router (screens), Table (task lists), Virtual (long logs and event feeds), Form (settings, split approval) | Adopt each only where it earns its place. |
| Graph | React Flow (xyflow) + ELK layout | The one non-TanStack, non-Zustand dependency. A DAG canvas is not worth hand-rolling. |
| Agent | Claude Code, headless, as a subprocess | `claude -p` with stream-json output, driven from Go. Hooks configured per session for enforcement. **Unverified**, M0 proves it. Fallback is the Anthropic Go SDK with our own loop. |
| Git | Shell out to `git` | go-git's worktree support is weak. |
| Watching | `fsnotify` | Test and file changes in the target repo. |

### Go to React contract

- **Commands and queries:** Wails-generated bindings. TanStack Query wraps the queries, and mutations invalidate by key.
- **Push:** Wails runtime events. A single `events` subscriber dispatches into the Query cache for domain changes and into Zustand for streaming output.
- **Rule:** if Go can reload it, it belongs in TanStack Query. If it is gone when you close the window and nobody would miss it, it belongs in Zustand.

## 4. Architecture

```
┌──────────────────────── Wails desktop app ────────────────────────┐
│  React UI (Zustand, TanStack)                                     │
│     │ bindings        ▲ runtime events                            │
│  ┌──▼─────────────────┴──────────────────────────────────────┐    │
│  │ Go                                                        │    │
│  │  app (Wails-bound façade, thin)                           │    │
│  │  domain  (types, state machine, invariants, pure)         │    │
│  │  store   (SQLite repos, migrations, event log)            │    │
│  │  engine  (phase orchestration, stage gate)                │    │
│  │  agent   (AgentDriver iface, claudecode impl, fake impl)  │    │
│  │  guard   (path ownership, hook endpoint)                  │    │
│  │  testrun (runner, result adapters, watcher)               │    │
│  │  gitwt   (worktrees, diff, commit suggestion)             │    │
│  │  export  (PRD and walkthrough markdown)                   │    │
│  └───────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘
             │ spawns                    ▲ PreToolUse hook (local socket)
             ▼                           │
        claude (headless)  ──────────────┘          target repo + .uwu/
```

### Repository layout

```
uwu/
├── main.go
├── app.go                 Wails façade
├── internal/
│   ├── domain/  store/  engine/  agent/  guard/  testrun/  gitwt/  export/
├── frontend/
│   └── src/
│       ├── routes/        TanStack Router
│       ├── features/      graph/  task/  grill/  split/  brief/  review/  settings/
│       ├── queries/       Query hooks over Wails bindings
│       ├── stores/        Zustand slices
│       └── lib/           events bridge, Result helpers
├── PLAN.md
└── docs/
```

### Go conventions

- Return `error` immediately, wrap with `%w`, custom error types for domain errors.
- `context.Context` on every long-running call, `errgroup` for concurrent work.
- `domain` has no I/O, so its tests are plain table tests.

### Frontend conventions

- `type` over `interface`. Strict TS. Branded IDs. Result types at the binding boundary instead of throwing.

## 5. Domain model

- **Project:** repo path, settings, test command config, status.
- **Task:** id, name, owner (`you` | `agent`), status (`not-started` | `in-progress` | `blocked` | `done` | `dropped`), dependencies (edges), contract, owned paths, tests, steps, open questions.
- **Stage:** one scaffold, work, close cycle for one human-owned task. Has a gate state.
- **Hint:** generated on demand, level 1 to 3, tied to a step and a diff snapshot.
- **Event:** append-only log of everything that changed. The audit trail and the UI's live feed.

Invariants (enforced in `domain`, not in UI):
- A task can't be `done` with red tests.
- The graph stays acyclic.
- A stub can't be wired in until its task is green.
- Only one human-owned stage is open at a time (the stage gate). With zero human-owned tasks this is trivially satisfied.
- Dropping a task with dependents requires resolving the dependents first.

## 6. Guidance settings (how much hand-holding)

The skill's options become first-class settings, layered: **app default, then project, then task override**.

| Dial | Values | Effect in uwu |
|---|---|---|
| Grill | `full` / `quick` / `skip` | Depth of the interview before the PRD. `skip` takes a pasted spec. |
| Ownership | `none` / `core` / `more` / `most` | How many nodes the split proposes for you. `none` proposes zero (see below). Per-task owner flips always allowed. |
| Hints | `none` / `graduated` / `full` | Whether the "stuck?" action exists, whether it escalates through three levels, whether a reference solution can be revealed. |
| Stop | `prd` / `plan` / `scaffold` | Where the engine pauses. |
| Level | `new` / `familiar` / `expert` | Step granularity and explanation depth. Overridable per task, since you may be expert in one area and new in another. |

**Presets** bundle the dials:

| Preset | grill | ownership | hints | stop | level |
|---|---|---|---|---|---|
| Guided | full | core | graduated | scaffold | new |
| Balanced (default) | full | core | graduated | scaffold | familiar |
| Independent | quick | more | none | plan | expert |

### Ownership `none`

`none` is specific to uwu. The `step-by-step` skill has no equivalent, so this value exists in the plan only. The user owns nothing, and every node belongs to the agent.

- **Split approval** still appears, with zero human nodes proposed. You can claim any node that hasn't started, at any time.
- **Stage gate:** it exists to serialize human stages, so with no human stage there is nothing to gate. The agent works the graph in dependency order and runs unblocked nodes in parallel (the M6 behavior becomes the default path).
- **Stop:** `prd` and `plan` still pause the engine. `scaffold` has no human stage to pause for, so the engine runs agent nodes to completion and halts at a final review (open decision 5).
- **Hints:** nothing to hint at, so the "stuck?" action is hidden. If you claim a node mid-run, the Hints dial applies to that node from then on.
- **Level:** only affects how much the briefs and the review summary explain.
- **Tests:** still written first and still required green before a node is `done` or its dependents start. They are now the main safeguard, since no human implements anything.
- **Write guard:** stays active with an empty path set. It becomes relevant the moment you claim a node.
- **Claiming an in-progress node:** the agent must stop first. The engine suspends that session, then flips the owner.

Possible later addition: let `level` adapt from your history on past tasks. That is not in scope until M7.

## 7. Skill capability to UI surface

| Skill capability | uwu surface |
|---|---|
| Grill (one question, a recommendation each) | Interview panel: question, recommended answer, accept or override. |
| PRD, task table, dependency tree | The graph is the primary view. A drawer holds problem, stories and decisions. |
| Split approval | Click a node to flip its owner. Diff against the proposed split. |
| Walkthrough | Task panel: resume-here banner, checkbox steps, contract and stub link. |
| Failing tests | Live red and green list, run from the app, updated by file watcher. |
| Graduated hints | "Stuck?" generates a hint from your current diff. Escalates nudge, pointer, example. |
| Stop and brief | Stage gate screen: what exists, where to start, test command, suggested commit. |
| Close stage and review | Review view: test results beside qualitative findings. |
| Resume, list, drift | Home screen is the list. Resume is opening the project. Drift cannot occur. |
| Options | Settings panel with preset picker and per-task overrides. |

## 8. Enforcement design

All of this is **unverified until M0**.

- **Write guard:** the agent session is started with a PreToolUse hook that calls back to the app over a local socket. If the target path is owned by an in-progress human task, the call is denied with a reason the agent can read.
- **Stage gate:** the engine refuses to start the next agent stage while a human stage is open, regardless of what the agent asks for.
- **Stub lock:** stubs throw their `TODO(you)` marker. A static check rejects any import of a stub from code that runs until its task is green.
- **Escape hatch:** "just do it" flips a task's owner and logs the event. It does not need regeneration machinery because walkthroughs are exports.

## 9. Milestones

Each ends with something you can run. The "You" column is the proposed human-owned work, to be confirmed in split approval.

| # | Milestone | Exit criteria | You own (proposal) |
|---|---|---|---|
| M0 | **Spike** | Wails v2/v3 decision. Drive `claude` headless from Go through gated phases. Hook blocks a write to a human-owned path. Test run streamed into a Wails event. | Hook and guard decision logic |
| M1 | **Domain and store** | Domain types, state machine, SQLite repos, migrations, event log, markdown export, importer for existing `prd/` folders. | State machine and invariants |
| M2 | **Read-only graph UI** | Wails shell, React shell, Query and Zustand wiring, graph with wave layout, task panel, live updates from events. | Graph layout and node design |
| M3 | **Planning flow** | Grill, PRD, split approval, walkthrough generation in the UI, driven by `AgentDriver`. | Split approval UX |
| M4 | **Stage loop** | Scaffold, stub, red tests, write guard, brief gate, close stage. Test adapters: generic command, `go test -json`, vitest JSON, JUnit XML. | Stage engine and guard |
| M5 | **Hints and review** | On-demand hints from your diff, review view, commit suggestion. | Hint prompt design |
| M6 | **Parallel agent branches** | Agent works unblocked agent-owned nodes in git worktrees while you work, with merge on completion and conflict surfacing. | Merge policy |
| M7 | **Polish** | Multi-project list, per-task level adaptation, fast path for small tasks, settings presets, packaging and signing for macOS. | Fast-path heuristics |

M6 is the differentiator and the riskiest piece, so it stays late. M4 is the core product.

## 10. Testing

Per the standards: mostly integration, few unit tests.

- **Go:** state machine, invariants and guard decisions get real tests. Store tests run against a temp SQLite file. The engine is tested against the `fake` `AgentDriver`, which replays scripted sessions.
- **Frontend:** a few Vitest tests for the events bridge and store slices. Playwright flows for the graph and the stage loop, run against `wails dev`'s browser endpoint with the real Go backend (**to verify in M0**).
- **Not tested:** layout, styling, presentational components.

## 11. Risks

| Risk | Mitigation |
|---|---|
| Hook-based enforcement is weaker than assumed | M0 exists to find out. If so, fall back to the Anthropic Go SDK with our own tool layer, where we control every write. |
| Headless Claude Code output or hook interface changes | Narrow `AgentDriver` interface, one implementation file, recorded-session fixtures. |
| Wails v3 instability or v2 limits | Pick in M0. Keep Wails-specific code in `app.go` and the events bridge only. |
| Target repos use any language, so test parsing varies | Generic command adapter first, then specific adapters per milestone M4. |
| Worktree merges in M6 produce conflicts with human edits | Agent branches only touch agent-owned paths. Conflicts become a task, not a silent merge. |
| Scope creep toward an omo clone | Section 1 non-goals. Revisit at the end of M4. |

## 12. Open decisions

1. Wails v2 vs v3 (M0).
2. Whether `.uwu/` is committed or git-ignored by default.
3. Whether the first agent implementation uses the CLI subprocess or the Go SDK (M0).
4. Whether the markdown export should be live-synced to an Obsidian vault or generated on demand.
5. With ownership `none`, where does the human review happen? Proposal: a final review gate before anything is called done. Alternative: per-node review as each finishes.

## 13. Building uwu with `step-by-step`

We dogfood the skill. Proposed settings for this project are in the conversation, pending your choices. Notes on how the skill's assumptions meet this repo:

- The skill writes to `../prd/uwu/<task>/`, one level above this repo. That is outside the working directory, so Claude Code will ask for permission.
- The skill finds the repo root with `git rev-parse --show-toplevel`. This directory is not yet a git repo, so it needs `git init` first.
- Commits are yours. The skill suggests checkpoints and never commits.
- The skill's `own` has three values (`core`, `more`, `most`). `none` is uwu-only, so building uwu itself uses one of those three.
