<table align="center">
  <tr>
    <td align="center">
      <h2>🚫 Abandoned</h2>
      <p><strong>Planned to build this <em>in April</em> — but Anthropic just shipped it as a first-class feature in Claude Code.</strong></p>
      <p>See <a href="https://code.claude.com/docs/en/agent-view"><strong>Agent View</strong></a> in the Claude Code docs.</p>
      <p><sub>The design notes below are preserved for posterity.</sub></p>
    </td>
  </tr>
</table>

---

# kitchen

A personal tool for orchestrating parallel coding-agent sessions.

> **Status:** abandoned (see banner above). The v1 spec lives at [`docs/superpowers/specs/2026-04-29-kitchen-design.md`](docs/superpowers/specs/2026-04-29-kitchen-design.md). No implementation.

## A note on the metaphor

The naming throughout this project leans on a restaurant-kitchen analogy: you are the **General Manager**, you talk to a **Kitchen Manager** who coordinates the line, and the actual work happens in **Workers** assigned to isolated **departments** (the stations of a kitchen — grill, pastry, salad, etc.). "Dropping in" to a worker is the GM walking the kitchen floor to watch a station live.

It's a loose, evocative metaphor — not a domain model. Don't read too much into it. It picks up where the cooking analogy stops being useful (e.g., workers don't have shifts, departments don't share ovens). If a future term ever forces the metaphor, prefer the clearer term over the cute one.

## What it solves

Running multiple Claude Code and Codex sessions in parallel across different repos and projects gets unwieldy fast. The pain isn't typing into tabs — it's:

- Not knowing which session needs attention
- Losing context every time you switch
- Parallel agents stepping on each other's repo state

## Mental model

```
        ┌───────────────────────┐
        │  You (General Manager)│
        └──────────┬────────────┘
                   │ talks to one thing
                   ▼
        ┌───────────────────────┐
        │   Kitchen Manager     │  Coordinates, never executes.
        │   (a Claude session)  │  Tells you what needs you.
        └──────────┬────────────┘
                   │ reads state of
                   ▼
        ┌────────────────────────────────────┐
        │  Workers (Claude Code, Codex, ...) │  Do the actual work.
        │  Each in its own workspace.        │  You can drop in to watch.
        └────────────────────────────────────┘
```

You talk to the **Kitchen Manager**. It knows the live state of every workstream and surfaces what needs you, ranked by urgency. Real work happens in **Workers** — heterogeneous, pluggable agent sessions that each run in their own isolated workspace ("department"). You can drop into any Worker's live view to watch token-by-token output or interject.

## Architecture at a glance

- **Backend daemon** (`kitchend`) — Bun + TypeScript. Single binary. HTTP + WebSocket on `127.0.0.1:8765`. Holds the event log, the in-memory bus, the supervisor, and the kitchen MCP server.
- **Event log** — append-only SQLite (WAL). Single source of truth for all worker activity. Every event has a monotonic `seq` that drives subscriptions and replay.
- **Workers** — pluggable. Each implements one minimal TypeScript interface (`start`, `sendInput`, `interrupt`, `kill`, `events`, `snapshot`). v1 ships `ClaudeCodeWorker` (via `@anthropic-ai/claude-agent-sdk`) and `CodexWorker` (via `@openai/codex-sdk`).
- **Workspaces** — every run gets an isolated working directory by default (`fresh` subdir, optional `git worktree`, or `inherit`). Prevents parallel agents from clobbering each other's repo state.
- **Kitchen Manager** — just a Worker with the kitchen MCP server attached and a coordinator-flavored system prompt. Configurable; you can run a Codex KM if you want.
- **Frontends** — TUI (Ink + React) and a native macOS app (SwiftUI). Both clients of the same backend API.

See the [design spec](docs/superpowers/specs/2026-04-29-kitchen-design.md) for the full architecture, schema, event types, and decision rationale.

## v1 scope (planned)

In:
- Backend daemon with HTTP+WS API, SQLite event log, in-memory bus
- Per-run workspaces (fresh + git worktree + inherit)
- `ClaudeCodeWorker` and `CodexWorker` with token-level streaming
- TUI with full controls (list, drop-in, send input, interrupt, kill, new run)
- macOS app with input parity (excluding worker creation UI, which stays CLI-only)
- Default KM bootstrap with `kitchen ask "<question>"` CLI
- Homebrew distribution

Out (interface-ready, deferred):
- Persistent always-on KM session
- Headless dispatch via SDK as a library
- macOS notifications surfacing KM alerts
- `launchd` auto-start
- Multi-user / cloud / auth

## Stack

| Layer | Choice |
|---|---|
| Backend daemon, CLI, TUI | Bun + TypeScript (single static binary) |
| macOS app | SwiftUI |
| Storage | SQLite (WAL) |
| Agent SDKs | `@anthropic-ai/claude-agent-sdk`, `@openai/codex-sdk` |
| Distribution | Homebrew (`brew install kitchen` + `--cask kitchen-app`) |

## Layout

```
.
├── docs/superpowers/specs/    # design specs
├── README.md                  # you are here
└── ...                        # (implementation TBD)
```

## License

Personal project. No license declared yet.
