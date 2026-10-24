# CLAUDE_REMEMBER

Handoff notes for a fresh Claude session (written 2026-10-24, moving to another machine). Read this file, then `PLAN.md`, then continue at "Where we left off".

## What uwu is

A desktop app for human-in-the-loop coding. A task graph is the main surface. Each node is owned either by the human or by the agent. The agent works its own nodes while the human works theirs, and the app enforces who touches what (write guard, stage gate, stub lock) instead of just asking the agent to behave.

The idea comes from two sources:

- **The `step-by-step` skill** (the user's own Claude Code skill). Its functionality is the model: grill, PRD, split approval, walkthroughs, scaffold-then-stop stages, red tests that define done, graduated hints, resumability. uwu turns that workflow into an app.
- **omo (https://omo.dev)**, an autonomous multi-agent coding tool, for UX only. Borrowed: graph laid out in waves, a status line per node, parallel threads, live progress. Not borrowed: its autonomy-first philosophy, multi-agent swarms, multi-model orchestration. This is based on the homepage only, and the product itself was never tried.

The user wants the human share to be configurable all the way down. Ownership `none` (the user owns nothing) is valid and exists in uwu only. The skill has no such value.

## The plan

`PLAN.md` in this directory is the source of truth for the product. It covers the stack, architecture, domain model, guidance settings (the hand-holding dials and presets), enforcement design, milestones M0 to M7, testing, risks, and open decisions.

Key decisions already made (also in `PLAN.md`):

- Desktop app: **Wails (Go) with a React frontend**. Zustand for client state, TanStack (Query, Router, Table, Virtual, Form) for the rest. React Flow plus ELK for the graph. Go is the backend, in-process, no separate server.
- State in SQLite (`modernc.org/sqlite`, pure Go) inside the target repo's `.uwu/`. Markdown PRDs and walkthroughs are exports, never inputs.
- Agent is behind a narrow `AgentDriver` interface. First implementation is Claude Code headless as a subprocess, with hooks for enforcement. **Unverified**, and M0 exists to prove it. Fallback is the Anthropic Go SDK with our own loop.
- Name is `uwu`. This directory is the repo root. No license for now.
- This directory is **not a git repo**. The skill falls back to the current directory, so `git init` is optional.

## The user's preferences

- Global instructions live in `~/.claude/CLAUDE.md` on the old machine. Copy them over, because they matter:
  - Two modes. **Dr. Prometheus** (kind, thorough teacher) handles open-ended questions. **Mr. Linux** (blunt, critical) handles reviews and specific tasks. A mode is switched by writing `(DP)` or `(ML)` early in a message, and each mode has an ASCII sign-off.
  - Standards: minimal comments, `type` over `interface` in TypeScript, Result types, wrapped errors with `%w` and `context.Context` in Go, mostly integration tests over unit tests.
- The user wants counterpoints raised, not agreement. They asked for pushback on the idea early on and accepted it.
- Commits are the user's. Never commit unless asked.

## The `step-by-step` run in progress

We are running the skill to produce a set of PRDs for uwu.

**Settings:** `grill=full`, `own=most`, `hints=graduated`, `stop=prd`, `level=familiar`.

- `own=most` means the user owns nearly everything. Claude does only boilerplate, setup, config, dependency installation, wiring, mechanical refactors and tests.
- `stop=prd` means Claude stops after the PRD is written and the owner split is approved. It does not write walkthroughs or scaffold yet.

**Decisions so far from the grill:**

1. **PRD structure: one PRD per milestone, M0 to M7, with a twist.** M0 gets a full PRD. M1 to M7 get short provisional PRDs, each with only the goal, dependencies and open questions. The reason is that M0 settles things that change everything after it: Wails v2 vs v3, whether hook enforcement works, and CLI subprocess vs Go SDK. The user is taking an agile approach and expects things to change as they test.

**Where we left off:** Question 2 was asked and is waiting for an answer.

> **Question 2: what happens to the M0 spike code afterwards?**
> - A. Throwaway in `spike/`, deleted after M0. Only decision records and recorded Claude session fixtures survive.
> - B. Seed. Structured as `internal/...` from the start with tests, and M1 builds on it.
> - C. Hybrid. The spike is throwaway, but the pieces that proved out (hook endpoint, session driver) are promoted into `internal/` with tests at the start of M1.
>
> Claude recommended **C**. The user has not answered yet.

Likely remaining grill branches (not yet asked): M0's definition of done and the decision criteria for each of its three questions, the test seams and test commands for M0, what the provisional PRDs for M1 to M7 must contain, open decision 5 in `PLAN.md` (where human review happens under ownership `none`), and where the user wants to learn versus just get it done (this calibrates the `own=most` split).

**No PRDs have been written.** The folder `/Users/meperalta/src/personal/prd/` does not exist yet. The skill puts PRDs at `<parent of repo>/prd/uwu/<task-name>/`, one level above this repo, outside its git. Tell the user once when creating it. It is meant to be opened as an Obsidian vault.

## How to resume

1. Read this file and `PLAN.md`.
2. Invoke the skill again with the same settings, for example `/step-by-step grill=full own=most hints=graduated stop=prd level=familiar resume`, or restate the goal. There is no task folder yet, so `resume` will find nothing. Start a new task and pick up the grill at Question 2, treating Question 1 as settled.
3. Ask Question 2 again if the user still hasn't answered. Grill one question at a time, each with a recommended answer.

## Setup on the new machine

The skill and the environment need checking, because the old machine had gaps.

- **The skill:** it lives in the user's dotfiles at `dots/stow-dir/claude/.claude/commands/step-by-step/`, stowed to `~/.claude/commands/step-by-step/`. Confirm it exists on the new machine. It contains `SKILL.md`, `README.md`, `references/` and `scripts/` (Python 3.12 standard library only).
- **Running the skill's scripts:** run them as `python3 <skill-dir>/scripts/<name>.py`, **without** `-I`. With `-I` the script can't import its sibling `_common.py`.
- **Toolchain missing on the old machine:** Go, the Wails CLI and pnpm were not installed. Present were Node v26.7.0, npm, Claude Code 2.1.309, git 2.54.0 and sqlite3. Check the new machine. Installing Go and Wails is a Claude-owned task under `own=most`, but ask the user before installing anything system-wide.
- **Permissions:** reading files outside the working directory, such as the skill's `SKILL.md` in the dotfiles directory, was denied once. Add the skill directory as an allowed directory, or the skill may behave differently than expected.
- **Web tools:** on the old machine WebFetch and WebSearch failed because their small helper model (`claude-3-5-haiku-20241022`) wasn't available, and `ANTHROPIC_MODEL` was pinned to `Opus 4`. A curl fallback worked. Check whether this reproduces on the new machine.

## Things not to assume

- The Claude Code hook and headless interface behavior in `PLAN.md` has not been tested. Treat §3 and §8 as hypotheses until M0.
- Wails v3's release status is unknown. Check current stability before choosing.
- omo's behavior is known only from its homepage.
