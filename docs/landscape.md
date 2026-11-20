# Landscape: open-source agent harnesses and orchestrators

**Last updated:** 2026-11-20 · **Research snapshot:** 2026-10-03 to 2026-10-06

This page collects the landscape research done while revisiting kitchen. The idea being revisited is a standalone, multi-provider harness. Its default mode is a Kitchen Manager that you talk to. The Kitchen Manager dispatches worker agents and watches them, each worker in its own git worktree. It shows you a ranked list of the workers that need you.

Supporting material on this branch:
- Full report, with citations: [`reports/Open source coding agent harnesses.md`](../reports/Open%20source%20coding%20agent%20harnesses.md)
- Per-topic research notes: [`research_notes/Open source coding agent harnesses/`](../research_notes/Open%20source%20coding%20agent%20harnesses/)
- Original v1 spec (shelved): [`docs/superpowers/specs/2026-04-29-kitchen-design.md`](superpowers/specs/2026-04-29-kitchen-design.md)

The original report was written before Paperclip and OpenRig were found. Where this page and the report disagree, this page is the newer view.

---

## TL;DR

1. **Don't build on Codex, and don't write a harness from scratch.** Every serious harness now exposes a machine protocol: Codex app-server, `opencode serve`, pi RPC, ACP. Harnesses have become workers you rent, not foundations.
2. **The parallel-agent dashboard is commoditized.** Claude Code Agent View, the Codex app, Orca, vibe-kanban, Emdash, Conductor and others already offer worktree-per-agent plus a status list.
3. **Two projects already ship most of kitchen.** This is newer than the original report.
   - **[OpenRig](https://github.com/mvschwarz/openrig)** (Apache-2.0, ~5.5k★) is almost exactly the kitchen v1 shape. It's a local daemon with SQLite, a TUI, a CLI and an MCP server. You talk to a lead agent, it has an attention surface, and it runs unmodified Claude Code and Codex sessions in tmux.
   - **[Paperclip](https://github.com/paperclipai/paperclip)** (MIT, ~98k★) is the heavyweight version, async and ticket-based. It has adapters for 13 harnesses, org charts, an inbox and approvals, budgets and worktrees.
4. **Recommendation: trial OpenRig first, then Paperclip, before writing any kitchen code.** Kitchen's remaining niche is narrow:
   - a full, replayable structured event log (rather than hook signals)
   - broader harness coverage through ACP
   - less invasive setup

   Each of these is plausibly an upstream contribution rather than a new project.

---

## 1. Harnesses (the worker layer)

| Project | License | Language | Providers | Embedding surface | Sandbox / permissions | Multi-agent | Fit for kitchen |
|---|---|---|---|---|---|---|---|
| **Codex CLI** (openai/codex) | Apache-2.0 (+CLA) | Rust (~118 crates) | OpenAI Responses API only (`wire_api="chat"` became a hard error on 2026-02-01). Built in: OpenAI, Bedrock, Ollama, LM Studio | `app-server` JSON-RPC (stdio/unix/ws) that sends approval requests to the client; `exec --json`; TS SDK wraps exec, so approvals never reach the client | OS sandbox (Seatbelt, bubblewrap, Windows); approval policies; "Guardian" reviewer | Stable `spawn_agent` tools (6 threads, depth 1, same provider); managed worktrees | **Worker** for OpenAI/ChatGPT-plan jobs via app-server. Not a base. |
| **opencode** (anomalyco, formerly sst) | MIT | TS on Bun, Effect-TS | ~20 AI SDK providers + models.dev; ChatGPT and Copilot login; Claude subscription login removed | `opencode serve` HTTP+SSE (OpenAPI 3.1), `opencode acp`, in-process sdk-next | allow/ask/deny glob rules; default `"*": "allow"`; no OS sandbox; 2026 CVEs | Subagents via `task` tool, background jobs; experimental worktree/workspace layer | **Worker**; **borrow** its event-sourcing schema |
| **pi** (earendil-works, formerly badlogic/pi-mono) | MIT | TS, Node ≥22.19 | ~35 providers + any OpenAI-compatible | `--mode rpc` JSONL, in-process SDK, libraries `pi-ai` / `pi-agent-core` | None by design ("YOLO") | Core leaves them out; experimental `pi-durable` task graph | **Library** (pi-ai, agent-core) + **worker** |
| **Gemini CLI** | Apache-2.0 | TS | Mostly Gemini | stream-json headless, native ACP | Sandbox, trusted folders | Subagents (Apr 2026), A2A remote agents | Worker via ACP |
| **Goose** (Linux Foundation AAIF) | Apache-2.0 | Rust | 15+ | ACP server *and* client | n/a | Parallel subagents, YAML recipes | Worker; borrow its pattern of driving other agents as providers over ACP |
| **Cline** | Apache-2.0 | TS | Many | Node SDK, headless NDJSON, `--acp` | Command guardrails | Parallel instances | Worker via ACP |
| **Qwen Code** | Apache-2.0 | TS (fork of Gemini CLI) | OpenAI, Anthropic, Gemini, Qwen, local | `qwen serve` HTTP+SSE, SDKs, ACP | n/a | Subagents, Agent Teams | Worker via ACP |
| **OpenHands SDK** | MIT | Python | Provider-agnostic | REST+WS agent server | Security analyzer; Docker workspaces | n/a | Architecture reference |
| **Crush** | FSL-1.1-MIT | Go | Many | `crush serve` | Prompts / `--yolo` | n/a | Reference (TUI, server split) |
| **Aider** | Apache-2.0 | Python | LiteLLM | `--message` scripting | Git auto-commit | None | Reference (repo map, edit formats) |
| Roo Code | — | — | — | — | — | — | Shut down 2026-05-15 (archived) |
| Kilo Code | MIT-ish (check) | TS (fork of opencode) | Kilo gateway | — | — | — | Use upstream opencode instead |

**ACP (Agent Client Protocol)** is the common language. About 38 to 50 agents support it natively, and Codex and Claude Code work through adapters. It standardizes `session/new|load|prompt|cancel|update|request_permission`. Kitchen-specific needs would go in `_kitchen/*` extension methods or `_meta` fields: worktree binding, model choice, cost, and structured done/blocked outcomes.

### Correction to the v1 spec

§11.2 of the v1 spec puts the Codex worker on `@openai/codex-sdk`. That SDK wraps the non-interactive `codex exec`, so approval requests never reach kitchen, and a Codex worker could never reach `awaiting_input`. The worker should drive **`codex app-server`** directly, using a pinned binary and generated TS types.

### Auth risks

- **Anthropic:** using Claude Pro/Max subscription logins in third-party harnesses is prohibited. Anthropic blocked third-party OAuth on 2026-01-09, wrote it into its docs on 02-19 and enforced it from 04-04. A harness that calls models directly needs Claude API keys. Running the real Claude Code CLI, as OpenRig does, avoids this.
- **OpenAI:** using ChatGPT plans in other tools is tolerated, but only on the strength of executive statements. There's no written policy. Driving the official `codex` binary is the lowest-risk route.

---

## 2. Libraries for building your own harness

| Library | Notes |
|---|---|
| **pi-ai + pi-agent-core** (MIT) | The best fit for coding agents. ~35 providers, partial-JSON tool streaming, cross-provider handoff of thinking blocks, a credential-store lock for OAuth refresh, steer/follow-up queues, and storage left to the host. Risks: Earendil acquired it in April 2026, 1.0 only shipped 2026-10-01, and outside PRs are auto-closed. |
| **Vercel AI SDK 7** | Broadest provider list. Has `ToolLoopAgent` (approval policies), `WorkflowAgent` (durable) and a sandbox abstraction. An experimental `HarnessAgent` wraps Claude Code, Codex and Pi. Officially Node 22+. |
| **Claude Agent SDK** | Claude only, under Anthropic Commercial Terms. The right way to build a Claude worker, not a harness core. |
| Mastra / LangGraph.js / OpenAI Agents SDK | Each brings its own orchestration model, which would duplicate the kitchen daemon. |

Practitioner consensus on a minimal coding-agent loop:
- 4 tools: read, write, exact-match edit, bash
- a system prompt under 1k tokens, kept as a stable cached prefix
- output clipping and compaction
- bounded subagents

### Things to borrow (designs or MIT/Apache code)

| Borrow | From |
|---|---|
| `event(aggregate_id, seq)` schema, versioned events, projectors, explicit replay after a seq | opencode `core/src/event`, `src/sync` |
| A stable prompt-cache prefix via "context epochs"; truncating oversized tool output | opencode `CONTEXT.md` |
| allow/ask/deny permission evaluation | opencode `permission/evaluate.ts` |
| Thread/Turn/Item vocabulary; approvals sent from server to client | Codex app-server |
| Worktree conventions | Codex `worktree` crate, opencode `worktree/index.ts` |
| Owned child tasks; background tasks that survive the parent being aborted | pi-durable (design only) |
| Heartbeat protocol, atomic task checkout, adapter package layout, falling back to ACP | Paperclip |
| Attention-item model (`action`/`update`, urgency, `unblocks`), queue wake ladder, hook-based activity relay | OpenRig |

---

## 3. Orchestrators (the layer kitchen lives in)

### Dashboards (the human dispatches)

Claude Code Agent View (Claude only; shipped 2026-05-11), the Codex app, Conductor, Sculptor, vibe-kanban (its company shut down in April 2026), claude-squad, Nimbalyst, Emdash, Agent Orchestrator, Bernstein, **Orca** (MIT, ~84k★, runs 40+ CLI agents in worktrees with "needs attention" alerts), GitHub Mission Control, and Antigravity's manager view. All offer worktree per agent plus a status list. In nearly all of them, the human is the one dispatching work.

### Paperclip — [paperclipai/paperclip](https://github.com/paperclipai/paperclip)

MIT · ~98k★ · Node + React + embedded Postgres (Drizzle) · very active (PRs past #15,000).

| Kitchen idea | Paperclip |
|---|---|
| Workers on any harness, any provider | 13 adapter packages: Claude Code, Codex, opencode, pi, Gemini CLI, Cursor (+Cloud), Hermes, Grok, Kimi, OpenClaw. The Claude and Codex adapters have ACP paths, and the Codex one also uses `codex app-server`. |
| A manager that dispatches work | Org chart of agents. A "CEO" agent delegates by creating child issues, and Paperclip wakes the parent when the children finish. |
| "What needs me" | Inbox state on issues; `request_confirmation` interactions raised on task threads; approval gates and review stages |
| Isolated workspaces | Optional git worktrees; remote sandbox providers (E2B, Daytona, Modal, Kubernetes, …) |
| Durable history | Durable activity records; sessions resume across reboots for adapters that support it |
| Extras | Budgets with hard stops, per-model cost, scheduled routines, skills, out-of-process plugins, multi-org, RBAC |

**How it differs:** coordination is asynchronous and ticket-based. An agent wakes on a heartbeat, claims a ticket with an atomic checkout, does a bounded piece of work, writes back to the ticket and exits. It's built to "run a company," with a web UI. Its Agent Chat is experimental. It's heavy: Node, Postgres and React.

### OpenRig — [mvschwarz/openrig](https://github.com/mvschwarz/openrig)

Apache-2.0 · ~5.5k★ · Node 22+ and tmux · mostly one person's project ("the system behind my AI civilization experiments").

| Kitchen v1 spec | OpenRig |
|---|---|
| Local daemon, HTTP API | Daemon on `:7433` (Hono routes) |
| SQLite WAL event store | `~/.openrig/openrig.sqlite`, WAL, migrations |
| Live subscriptions | SSE at `/api/activity/events` |
| TUI + CLI + KM tools | TUI, `rig` CLI, MCP server (`rig mcp serve`), all clients of the daemon |
| Kitchen Manager you talk to | A lead agent that coordinates specialists; a built-in "kernel" operator for setup |
| "What needs me" | `attention` surface with `action`/`update` items, urgency, an `unblocks` field and a recipient |
| Workers | **Unmodified** Claude Code, Codex, Pi and Oh My Pi sessions in tmux; teams defined in YAML (`starter`, `workshop`, `factory`) |
| Drop in | `tmux attach` shows you the real CLI, live |
| Durable work | Queue with pickup, recovery, stuck-task sweeps, a wake ladder and a transition log |

**The design choice that cuts against the report:** OpenRig drives real CLIs instead of protocols.
- **Upside:** Claude subscription logins work normally, because the workers are genuine Claude Code. Live drop-in comes free.
- **Downside:**
  - The structured log is thin. The hook relay deliberately leaves out prompt text and tool arguments, so there's no full transcript to replay.
  - Setup is invasive. It writes trust entries, hooks and permission defaults into `~/.claude.json`, `~/.codex/config.toml` and workspace settings.
  - It covers fewer runtimes: no opencode, no Gemini CLI, no ACP.

### Positioning

| | OpenRig | Paperclip | Kitchen (revisited) |
|---|---|---|---|
| Shape | Personal, local, terminal | Team or company, web, async tickets | Personal, local, terminal |
| Manager | Lead agent you talk to | CEO agent delegating through issues | KM you talk to |
| Workers | Real CLIs in tmux | 13 adapters (CLI, ACP, app-server, HTTP) | Protocol adapters + ACP |
| Event fidelity | Lifecycle hooks | Activity records + run logs | Full replayable structured log (goal) |
| Claude subscription | ✅ (real CLI) | Via the real CLI adapters | API keys if it calls models directly |
| Status | Shipping | Shipping, 98k★ | Design only |

---

## 4. If kitchen is built in Rust instead of TS/Bun

The main recommendation doesn't change: workers are separate processes, so the language doesn't matter for them. What changes is which libraries kitchen can use in-process:

| Project | TS/Bun | Rust |
|---|---|---|
| Codex | Worker via app-server | Worker plus pinned crates (sandbox, `apply-patch`, `worktree`, `app-server-protocol` types) |
| Goose | Worker via ACP | Worker; possible provider-layer library (needs checking) |
| pi | Library + worker | RPC worker; design reference only |
| Vercel AI SDK 7 | Alternative library | Not usable |
| opencode | Worker; copy event-log code | Worker; port the event-log design |
| Claude Agent SDK | In-process Claude worker | Not usable; drive the Claude Code CLI or ACP |
| ACP | TS SDK | Rust SDK (Zed, which created ACP, is written in Rust) |

- **Gains:** Codex's sandbox and patch crates, a single binary, ratatui for the TUI, and no Bun-compatibility risk.
- **Losses:** the ready-made multi-provider agent library. The Rust options weren't researched: Goose's crates, `rig`, `genai`, and Codex's OpenAI-only client.

---

## 5. Where this leaves kitchen

1. **Trial OpenRig** on a real repo. Its `starter` team, a Claude builder plus a Codex reviewer, takes an afternoon to set up.
2. **If OpenRig fits,** kitchen becomes upstream contributions:
   - structured event capture
   - ACP and opencode runtimes
   - lighter setup
3. **If async tickets suit you better than a live conversation,** use Paperclip, and build anything missing as a plugin or a TUI client.
4. **Build kitchen standalone only if** both leave a real gap. Even then, start from the borrow table above.

### Open questions

- Bun behavior of pi-ai, AI SDK 7 and the Claude Agent SDK.
- Whether Codex subagents can be fully turned off for each thread from outside.
- Memory cost per running `opencode serve` instance.
- Maturity of the Rust LLM/agent crates.
- How well OpenRig's hook-only event capture holds up once there are several workers.
