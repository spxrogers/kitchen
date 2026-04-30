# Kitchen — Design Spec

**Date:** 2026-04-29
**Status:** Approved (pending user review)
**Author:** steven (with Claude as design partner)

---

## 1. Problem & Mental Model

steven runs multiple Claude Code sessions in parallel across different repos and projects, plus other workstreams. The pain isn't typing into tabs — it's **not knowing which session needs attention**, and **losing context every time he switches**.

Mental model: steven is the General Manager. He wants to talk to one thing — a **Kitchen Manager** (KM) — that knows the state of every workstream and tells him what needs him. The KM coordinates; it never executes. Individual **Workers** do the work. He can also "drop in" to any Worker's live view to watch or interject.

## 2. Design Goals (firm)

- **Multi-agent out of the box.** Claude and Codex both first-class. The Worker abstraction makes a third (Gemini, custom SDK script) a small addition, not a refactor.
- **Real-time streaming is v1.** Dropping into a Worker view shows live token-level output as if in a direct Claude Code session. KM gets live state, not polled snapshots.
- **Two frontends from day one.** TUI and macOS app, both reading from the same backend. Forces the backend to be a real API, not a frontend appendage.
- **Headless dispatch via SDK is post-v1, but interfaces make it a one-line addition.**

## 3. Non-Goals (v1)

- Multi-user / cloud sync
- Headless worker dispatch (interface ready, implementation deferred)
- Cross-worker peer messaging
- Auth / permissions
- Mobile clients
- Worker creation UI in macOS app (CLI-only)

---

## 4. Stack

| Layer | Choice | Rationale |
|---|---|---|
| Backend daemon | **Bun + TypeScript** | Single-binary distribution via `bun build --compile`, ~30ms cold start, mature SDK ecosystem (Claude + Codex), trivial homebrew formula. |
| TUI | **Ink + React** (TS) | Same language as backend; shares types; ships in same binary. |
| macOS app | **SwiftUI native** | "macOS native" taken at face value. Talks to backend over WS+JSON. |
| Storage | **SQLite + WAL** (`bun:sqlite`) | Stdlib-equivalent, ACID, embedded, all queries indexed. |
| Distribution | **Homebrew** | `brew install kitchen` (CLI/daemon) + `brew install --cask kitchen-app` (macOS .dmg). |

**Rejected**:
- Python: more complex homebrew formula (virtualenv_install + per-dep `resource` blocks), ~200ms CLI cold start, weaker Codex SDK story (Python SDK is editable-install only).
- Go / Rust: no `claude-agent-sdk` equivalent; would require building agent loop from scratch.
- Tauri/Electron for macOS: not native.

**Cross-language boundary**: SwiftUI ↔ TS backend communicates via JSON over WS+HTTP. Generate Swift structs from a JSON Schema at build time (one-off codegen script). No runtime FFI.

---

## 5. Architecture Overview

```
┌─────────┐     ┌─────────┐     ┌──────────┐
│   TUI   │     │  macOS  │     │  CLI     │
│ (Ink)   │     │ (Swift) │     │ (kitchen)│
└────┬────┘     └────┬────┘     └────┬─────┘
     │               │                │
     │  HTTP+WS over 127.0.0.1:8765   │
     ├───────────────┴────────────────┤
     │                                │
     ▼                                ▼
┌──────────────────────────────────────────┐
│          kitchend (Bun daemon)           │
│  ┌────────────────────────────────────┐  │
│  │   HTTP API (Bun.serve)             │  │
│  │   WebSocket subscriptions          │  │
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │   Supervisor                       │  │
│  │   ├── seq counter (in-mem)         │  │
│  │   ├── In-Memory Bus (EventEmitter) │  │
│  │   └── SQLite write queue (50ms)    │  │
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │   Workers (one process per type)   │  │
│  │   ├── ClaudeCodeWorker (SDK)       │  │
│  │   ├── CodexWorker (SDK)            │  │
│  │   └── (future) ManualWorker, ...   │  │
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │   Kitchen MCP Server (in-process)  │  │
│  │   exposed to KM workers            │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
                    │
                    ▼
              ┌──────────┐
              │ SQLite   │
              │ (WAL)    │
              └──────────┘
```

**Key invariants:**
1. Workers emit events; the Supervisor assigns `seq`, fans out to bus, writes to SQLite.
2. The bus is the live source. SQLite is the durable record. Bus has no history.
3. Subscribers replay from SQLite on attach, then tail the bus.
4. WS backpressure → disconnect → client reconnects with `sinceSeq`. Replay closes the gap.

---

## 6. Worker Interface

The single most load-bearing artifact. The interface is an event-emitting state machine.

### 6.1 Lifecycle

```typescript
type LifecycleState =
  | "created"        // instantiated, no run started
  | "starting"       // run requested, not yet streaming
  | "running"        // actively producing events
  | "awaiting_input" // user response required
  | "blocked"        // running but stuck (rate limit, tool permission, etc.)
  | "done"           // run completed successfully
  | "failed"         // run errored
  | "cancelled";     // user-stopped
```

Flat enum. No nested states. `blocked` substate reasons (rate-limited, tool-permission, network) are visible in the latest events but not in the snapshot.

### 6.2 Methods

```typescript
interface Worker {
  readonly id: string;
  readonly type: string;        // 'claude_code' | 'codex' | ...
  readonly label: string;

  start(task: TaskSpec): Promise<RunId>;
  sendInput(text: string): Promise<void>;
  interrupt(): Promise<void>;   // stop generating, stay alive for next input
  kill(): Promise<void>;        // terminate worker process

  events(): AsyncIterable<WorkerEvent>;  // LIVE-ONLY (no replay)
  snapshot(): WorkerSnapshot;
}

interface TaskSpec {
  prompt: string;
  context?: Record<string, unknown>;  // worker-type-specific overrides
  cwd?: string;
  model?: string;
}
```

**Rules:**
- `events()` is live-only. Replay is the Supervisor's responsibility, sourced from SQLite.
- One run at a time per Worker. Concurrency lives at WorkerPool level (creating multiple Workers).
- Worker-instance config (API key, default model, MCP servers, system prompt) lives in the constructor; per-task overrides go in `TaskSpec`.
- Workers do NOT persist, render, calculate priority, or retry. Those are upstream concerns.

### 6.3 Events emitted

```typescript
type WorkerEvent =
  | { type: "run.started";          runId: string; task: TaskSpec }
  | { type: "run.ended";            runId: string; reason: "done" | "failed" | "cancelled"; metadata?: Record<string, unknown> }
  | { type: "state.changed";        from: LifecycleState; to: LifecycleState; reason?: string }
  | { type: "output.text_delta";    runId: string; role: "assistant" | "user" | "system"; content: string }
  | { type: "output.message";       runId: string; role: "assistant" | "user" | "system"; text: string }
  | { type: "tool.call_started";    runId: string; toolId: string; name: string; argsRaw: unknown }
  | { type: "tool.call_progress";   runId: string; toolId: string; deltaContent: string; stream?: "stdout" | "stderr" }
  | { type: "tool.call_completed";  runId: string; toolId: string; ok: boolean; resultRaw: unknown }
  | { type: "input.requested";      runId: string; prompt?: string }
  | { type: "input.received";       runId: string; text: string }
  | { type: "meta";                 runId: string; key: string; value: unknown }
  | { type: "log";                  runId?: string; level: "debug" | "info" | "warn" | "error"; message: string };
```

`argsRaw` and `resultRaw` are `unknown` on purpose. **No normalization across worker types.** Frontends and KM render per-`worker_type`.

### 6.4 Worker snapshot (derived)

```typescript
interface WorkerSnapshot {
  id: string;
  type: string;
  label: string;
  lifecycleState: LifecycleState;
  currentRunId: string | null;
  attentionRequired: boolean;          // true when state ∈ {awaiting_input, blocked, failed}
  attentionReason: "awaiting_input" | "blocked" | "error" | null;
  attentionSinceMs: number | null;     // ts of state change into current attention state
  lastEventAtMs: number;
  currentTaskSummary: string | null;   // auto from TaskSpec.prompt; overridable via set_task_summary
}
```

Computed on demand from the events log. Cached in process; invalidated on relevant events.

---

## 7. Event Log Schema

Three tables. SQLite + WAL mode + `synchronous=NORMAL`.

```sql
-- Durable worker identity. One row per Worker, reused across runs.
CREATE TABLE workers (
    id          TEXT PRIMARY KEY,
    type        TEXT NOT NULL,
    label       TEXT NOT NULL,
    config      TEXT NOT NULL,       -- JSON: model, cwd, MCP servers, system prompt, persistTextDeltas, etc.
    created_at  INTEGER NOT NULL,
    archived_at INTEGER
);

-- Append-only event log. Single source of truth for activity.
CREATE TABLE events (
    seq        INTEGER PRIMARY KEY AUTOINCREMENT,  -- monotonic; subscription cursor
    ts_ms      INTEGER NOT NULL,
    worker_id  TEXT NOT NULL REFERENCES workers(id),
    run_id     TEXT NOT NULL,                       -- '' for non-run events
    type       TEXT NOT NULL,
    payload    TEXT NOT NULL                        -- JSON; schema validated by type
);
CREATE INDEX idx_events_worker ON events(worker_id, seq);
CREATE INDEX idx_events_run    ON events(run_id, seq);
CREATE INDEX idx_events_type   ON events(type, seq);

-- Run summaries. Materialized for cheap "list runs" queries; rebuildable from events.
CREATE TABLE runs (
    id            TEXT PRIMARY KEY,
    worker_id     TEXT NOT NULL REFERENCES workers(id),
    started_at_ms INTEGER NOT NULL,
    ended_at_ms   INTEGER,
    state         TEXT NOT NULL,
    task_summary  TEXT
);
CREATE INDEX idx_runs_worker ON runs(worker_id, started_at_ms DESC);

-- Singleton settings.
CREATE TABLE system_config (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
```

### 7.1 Decisions

- **`seq` = `INTEGER PRIMARY KEY AUTOINCREMENT`.** Monotonic; reuse-prevented by `AUTOINCREMENT`.
- **Counter lives in Supervisor process memory.** Initialized at boot from `SELECT MAX(seq) FROM events`. Atomic increments; assigned before bus emit and SQLite write.
- **`payload` is JSON TEXT.** No JSONB indexes in v1. Add expression indexes later if KM search needs them.
- **`run_id = ''`** for non-run events (e.g., worker-creation `state.changed`).
- **Same DB file** for all tables. Joins are easy; one file backup.
- **`text_delta` persistence is per-worker config (`WorkerConfig.persistTextDeltas`, default `true`).** Live streaming always works; only DB persistence is gated. Lets users with disk concerns toggle off without losing UX.
- **No retention/archival in v1.** ~10GB/year of events on local disk is acceptable.

### 7.2 Crash & shutdown

- **Crash**: lose ≤50ms of un-flushed events from the SQLite write queue. Acceptable.
- **Graceful shutdown** (firm requirement): SIGTERM/SIGINT or `POST /v1/shutdown` →
  1. Stop accepting new HTTP/WS connections
  2. Drain SQLite write queue completely
  3. Close WS connections with `code: "shutdown"`
  4. Close DB connection
  5. Exit

---

## 8. Storage + Live-Watch

### 8.1 Pattern

```
Worker.events() ──► Supervisor ──► assign seq ──┬──► InMemoryBus.emit(event)  ──► WS subscribers
                                                 └──► SQLiteWriteQueue (batched)
```

- **Bus**: single global `EventEmitter`, one event name (`"event"`). Subscribers filter by topic + seq.
- **SQLite writes**: queued, flushed every 50ms or every 200 events, whichever first. One `BEGIN IMMEDIATE` per flush.
- **No bus history.** Replay always from SQLite.

### 8.2 `sinceSeq` subscription algorithm

```
1. Subscribe to bus first (start buffering live events with their seq).
2. Read SQLite WHERE seq > sinceSeq (ordered by seq).
3. Send SQLite results in order.
4. Drain buffer; drop any seq <= max(SQLite seq sent). Send the rest in order.
5. Continue forwarding live bus events.
```

Race-free: bus subscription opens before DB read, so any concurrent writes are captured.

### 8.3 Backpressure

WS subscriber's `socket.bufferedAmount > 1MB` → disconnect with `code: "lagging"`. Client reconnects with `sinceSeq=lastSeqReceived`. Replay catches them up.

---

## 9. Backend API

`Bun.serve()` on `127.0.0.1:8765` (configurable port). All endpoints under `/v1/` prefix. JSON.

### 9.1 HTTP

```
# Workers
GET    /v1/workers
GET    /v1/workers/:id
POST   /v1/workers                       { id?, type, label, config }
PATCH  /v1/workers/:id                   { label?, config? }
DELETE /v1/workers/:id                   (soft-archive)

# Runs
GET    /v1/runs?worker_id=&state=&limit=
GET    /v1/runs/:id
POST   /v1/workers/:id/runs              TaskSpec → { runId }   (returns IMMEDIATELY)
POST   /v1/workers/:id/input             { text }
POST   /v1/workers/:id/interrupt
POST   /v1/workers/:id/kill

# Events (replay)
GET    /v1/events?worker_id=&run_id=&since_seq=&limit=

# KM
POST   /v1/km/ask                        { question, to? } → streams answer

# System
GET    /v1/health
POST   /v1/shutdown
GET    /v1/system_config/:key
PUT    /v1/system_config/:key            { value }
```

### 9.2 WebSocket

Single endpoint `ws://127.0.0.1:8765/v1/stream`. Multiplexed.

```typescript
type ClientMsg =
  | { type: "subscribe";   subId: string; topic: Topic; sinceSeq?: number }
  | { type: "unsubscribe"; subId: string };

type Topic =
  | { kind: "worker"; workerId: string }
  | { kind: "run";    runId: string }
  | { kind: "global" }                    // state.changed only (cheap firehose)
  | { kind: "all" };                      // every event (KM debug only)

type ServerMsg =
  | { type: "event";      subId: string; event: WorkerEvent; seq: number }
  | { type: "error";      subId: string; code: string; message: string }
  | { type: "subscribed"; subId: string }
  | { type: "completed";  subId: string };  // run ended; sub auto-closes
```

### 9.3 Versioning

`/v1/` prefix mandated; no plans for `/v2/`. If we ever need a breaking change, prefer additive evolution within `/v1/`.

---

## 10. Kitchen Manager

**KM is a Worker.** No special class, no special path. The "KM-ness" is configuration:
- Kitchen MCP server attached
- KM system prompt baked into worker config

### 10.1 Default-target indirection

`system_config.default_ask_target` holds the worker_id of the default KM. CLI sugar:

- `kitchen ask "<q>"` → dispatches a run on `default_ask_target`
- `kitchen ask --to <id> "<q>"` → overrides

Bootstrap on first daemon start: create one Claude-type KM worker (`id="km-claude-default"`, kitchen MCP server attached, KM system prompt as system instruction), set `default_ask_target` to it.

To switch to a Codex KM: create a Codex worker with kitchen MCP + KM prompt, then `PUT /v1/system_config/default_ask_target { value: "<new-worker-id>" }`. Multiple KM-flavored workers can coexist; only one is the default.

### 10.2 Kitchen MCP server (KM's tools)

In-process MCP server hosted by the daemon. Exposed to whichever workers list it in their config.

```
list_workers()                               → WorkerSnapshot[]
get_worker(id)                               → WorkerSnapshot + last 20 events summarized
tail_events(workerId, sinceSeq?, limit=50)   → raw Event[]
search_events({ query?, type?, workerId? })  → Event[]   (LIKE-based v1; FTS5 v2)
notify_user(message, urgency)                → writes to global:km_notifications topic
set_task_summary(workerId, summary)          → writes meta event { key: "task_summary", value: summary }
```

Post-v1: `dispatch_worker(spec)` → creates a new run programmatically.

### 10.3 Sync model

KM is **purely reactive** in v1. Each user turn calls `list_workers()` first; gets fresh snapshot. No background polling, no proactive notifications from the KM itself.

The backend writes attention-state alerts directly to `global:km_notifications` on relevant transitions (e.g., a worker entering `awaiting_input` for >N minutes). Frontends surface those. The KM is invoked when steven wants synthesized briefings.

### 10.4 KM model & system prompt

- **Default model**: Claude Sonnet (4.6 or current). Configurable per-KM.
- **System prompt sketch** (lives in KM worker's config; user can edit via `PATCH /v1/workers/:km-id`):

```
You are the Kitchen Manager for steven's parallel workstreams.

Your role:
- Coordinate, do not execute work. If asked to do work, refuse and offer to dispatch a worker
  (post-v1) or tell steven who should do it.
- On every turn, call list_workers FIRST. Never invent state.
- Identify workers requiring attention: state in {awaiting_input, blocked, failed}.
- Rank by attentionSinceMs ascending — longest-suffering first.
- Be terse and scannable. Use compact tables or bullet lists, not prose.

Format example for "what needs me?":
1. backend-claude (45m) — awaiting input on auth migration approach
2. frontend-codex (12m) — blocked: rate limit, will retry at 14:32
3. docs-claude  (3m)  — failed: tool permission denied for git push

Do not editorialize. Do not give opinions on quality of work. Status only.
```

### 10.5 Primary KM question

**"Which workers need me right now?"**

Operationalized: `WorkerSnapshot[]` filtered to `attentionRequired=true`, sorted by `attentionSinceMs` ascending. Drives the snapshot schema; everything else flows from this.

---

## 11. Worker Implementations

### 11.1 ClaudeCodeWorker

Uses `@anthropic-ai/claude-agent-sdk` directly. **No PTY-wrapping the CLI. No hooks-only.**

| Claude Agent SDK message | → kitchen `WorkerEvent` |
|---|---|
| `text_delta` blocks | `output.text_delta` |
| Assistant message complete | `output.message` |
| `tool_use` block | `tool.call_started` (raw `input` → `argsRaw`) |
| User message with `tool_result` | `tool.call_completed` |
| Permission request (via SDK hook) | `input.requested` with prompt |
| `result` event | `run.ended` (usage as metadata) |
| Internal session state | `state.changed` |

Session resumption: lean on SDK's `resume: sessionId`. Worker tracks current sessionId per run.

Permission mode: per-Worker config (`default` | `acceptEdits` | `bypassPermissions`). Default `default` (safe).

Auth: `ANTHROPIC_API_KEY` from env. Shared across workers.

### 11.2 CodexWorker

Uses `@openai/codex-sdk` (official, on npm).

| Codex event | → kitchen `WorkerEvent` |
|---|---|
| `thread.started` | `run.started` (thread_id → runId) |
| `turn.started` | `state.changed` → running |
| `item.started` (command_execution, file_change, web_search, etc.) | `tool.call_started` |
| `item/agentMessage/delta` | `output.text_delta` |
| Tool progress (stdout/stderr) | `tool.call_progress` |
| `item.completed` (type=agent_message) | `output.message` |
| `item.completed` (other types) | `tool.call_completed` |
| `turn.completed` | `state.changed` → awaiting_input or done |
| `thread.archived` | `run.ended` |

Sandbox/permission mode: per-Worker config; carries native Codex string (`untrusted` | `on-request` | `never` | `full-auto`). No fake common enum across worker types.

Auth: `OPENAI_API_KEY` or ChatGPT browser auth. Shared across workers.

### 11.3 Convergence vs divergence

**Above the Worker line** (frontends, KM, event log): all worker types are interchangeable.

**Below the line** (worker-internal mapping): tool names, permission strings, and tool args/results are preserved per-native shape. Frontends dispatch on `worker.type` for rendering.

---

## 12. Frontends

### 12.1 TUI (`kitchen tui`)

- Ink + React, ships in same binary as daemon and CLI.
- Worker list pane (left), sorted by `attentionRequired DESC, attentionSinceMs ASC, lastEventAtMs DESC`.
- Drop into worker view → live stream of all events.
- Send input when worker is `awaiting_input`.
- `Ctrl+I` interrupt, `Ctrl+K` kill.
- Tab to swap between workers; ESC returns to list.

### 12.2 macOS app

- SwiftUI native, distributed as `.dmg` via homebrew cask.
- Sidebar: worker list (same sort).
- Detail pane: live stream.
- Full input parity with TUI: send input, interrupt, kill, "new run" (with prompt entry).
- Worker creation/config UI is **not** in v1 (CLI only — config-heavy form, low frequency).
- `notify_user` events are subscribed but not surfaced as macOS notifications in v1 (defer).

### 12.3 CLI

- `kitchen daemon start | stop | status` — explicit control.
- Auto-spawn: `kitchen ask` and `kitchen tui` spawn the daemon if not running.
- `kitchen ask "<q>" [--to <id>]` — KM query.
- `kitchen tui` — launch TUI.
- `kitchen workers list | create | archive` — minimal worker management.

---

## 13. v1 Scope Summary

### IN v1

- Backend daemon: HTTP+WS, SQLite log, in-memory bus, Supervisor, graceful shutdown
- Kitchen MCP server (in-process)
- ClaudeCodeWorker
- CodexWorker
- TUI with full controls
- macOS app with full input parity (excluding worker creation UI)
- Default KM bootstrap (Claude type) with `kitchen ask` CLI
- Auto-spawn daemon
- Homebrew distribution (CLI + macOS cask)

### OUT v1 (interface-ready, deferred)

- ManualWorker
- `dispatch_worker` MCP tool (KM dispatching new runs)
- Worker creation/config UI in macOS app
- Persistent always-on KM session with diff-since-last
- Headless dispatch via SDK as a library
- macOS notifications for `notify_user`
- launchd auto-start
- Scheduled "morning briefing"
- FTS5 search in `search_events`
- Multi-user / cloud / auth / peer messaging

### Build order

Each step ends with a working demo.

1. **Foundations**: schema + bus + Supervisor + WS subscription mechanics
2. **HTTP API**: CRUD + dispatch endpoints
3. **ClaudeCodeWorker**: proves Worker shape; first end-to-end flow
4. **TUI v1**: proves live-stream UX (steven has a usable product here)
5. **KM bootstrap + kitchen MCP server + `kitchen ask`**: design-goal #4 satisfied
6. **CodexWorker**: proves abstraction across two SDKs
7. **macOS app v1**: design-goal #3 satisfied; v1 done

---

## 14. Open Items / v1.1 Backlog

Captured here so they don't get lost:

- ManualWorker (validates Worker interface against non-LLM use)
- macOS worker creation UI
- Persistent KM session with diff-since-last via `list_workers` enhancement
- macOS notifications for `notify_user` events
- launchd integration for daemon
- Scheduled KM briefings (cron-style)
- FTS5 indexes on event payloads
- `dispatch_worker` MCP tool (programmatic dispatch)
- "Snooze worker" affordance to suppress over-reporting on intentionally-paused workers
- Per-KM topical scopes (e.g., `kitchen ask --to docs-km`)

## 15. Known Acceptable Risks (v1)

- **Crash loses ≤50ms of un-flushed events.** Live subscribers saw them; durable history is missing them. Acceptable for personal tool.
- **Over-reporting on intentionally-paused workers.** A worker paused for legit reasons (you're thinking) shows as needing attention. Mitigation deferred (snooze affordance, paused substate).
- **macOS app cannot create workers.** CLI-only for v1. Acceptable for first ship.
- **No FTS5.** `search_events` is LIKE-based; slow on large logs. Add FTS5 if KM searches become slow.
- **No persistent KM context.** Each `kitchen ask` invocation starts fresh. Acceptable trade-off vs token burn.
