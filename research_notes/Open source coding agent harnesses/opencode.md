# opencode (anomalyco/opencode, formerly sst/opencode) — evaluation for kitchen

Method note: most architecture claims below come from a shallow clone of the repo taken on 2026-10-03 (HEAD `907b3bc`, committed 2026-10-02, monorepo version `1.18.34`). Source links point at `github.com/anomalyco/opencode/blob/dev/...`. Treat the paths as current only as of that commit, because the codebase is mid-refactor (see churn below). "R" = repo path.

## 1. Identity, license, runtime, popularity, cadence, governance

### Takeaway
opencode is an MIT-licensed TypeScript/Bun monorepo with about 212k stars. It is backed by Anomaly Innovations (the former SST team), lives at github.com/anomalyco/opencode, and ships patch releases roughly every 2–5 days. The old Go TUI is gone: the TUI is now TypeScript, built on Anomaly's own OpenTUI with SolidJS.

### Cited Findings
- The repo moved from `sst/opencode` to `anomalyco/opencode`. The README badges and the `brew install anomalyco/tap/opencode` line point at anomalyco, and `git clone github.com/sst/opencode` still resolves (redirect). — [R: README.md](https://github.com/anomalyco/opencode/blob/dev/README.md); the GitHub org was renamed and sst/opencode now redirects; "same project" — [innfactory / search summary](https://innfactory.ai/de/ki-harness/opencode/), [bmannconsulting note on Anomaly](https://bmannconsulting.com/notes/anomaly/)
- The company is Anomaly Innovations (Jay V CEO, Frank Wang CTO). Its portfolio covers SST, OpenNext, OpenAuth, OpenTUI and opencode. — [horadecodar: SST & OpenCode](https://horadecodar.com.br/sst-opencode/), [aiwiki opencode](https://aiwiki.ai/wiki/opencode) (secondary; aiwiki returned 429 on fetch, claims seen via search snippet only)
- License is MIT ("Copyright (c) 2025 opencode"). — [R: LICENSE](https://github.com/anomalyco/opencode/blob/dev/LICENSE)
- GitHub stats on 2026-10-03: about 211.6k stars, 28.1k forks, 15,847 commits on the default branch `dev`. — [GitHub repo page](https://github.com/anomalyco/opencode)
- Release cadence: v1.18.34 (Sep 30 2026), .33 (Sep 28), .32 (Sep 21), .31 (Sep 14), .30 (Sep 9), .29 and .28 (both Sep 4), .27 (Sep 2), .26 (Sep 1), .25 (Aug 28). The releases list has 88 pages (about 880 releases). — [GitHub releases](https://github.com/anomalyco/opencode/releases)
- Recent release notes are mostly provider plumbing: GPT-6 / "gpt-6-astra" prompt, Claude 5 thinking-block tolerance, Copilot adaptive thinking, Bedrock fixes, ACP session model restoration (v1.18.31), and "Send namespaced session and parent-session identity headers" (v1.18.34). — [GitHub releases](https://github.com/anomalyco/opencode/releases)
- Language and runtime: Bun + TypeScript (`bun.lock`, `bunfig.toml`, turbo). It uses the Effect library heavily (`effect` dependency; services are `Context.Service` / `Layer`). The `#db` import switches between `db.bun.ts` and `db.node.ts`, so Node is also supported. — [R: packages/opencode/package.json](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/package.json)
- The TUI is `packages/tui` and is written in TSX with `@opentui/core`, `@opentui/solid`, `@opentui/keymap` and `solid-js`, so there is no Go TUI. — [R: packages/tui](https://github.com/anomalyco/opencode/tree/dev/packages/tui)
- Monorepo packages: app, cli, client, codemode, console, containers, core, desktop, docs, effect-drizzle-sqlite, effect-sqlite-node, enterprise, function, http-recorder, httpapi-codegen, identity, llm, opencode, plugin, protocol, schema, sdk, sdk-next, server, session-ui, slack, stats, storybook, tui, ui, web. — [R: packages/](https://github.com/anomalyco/opencode/tree/dev/packages)
- Distribution: `curl opencode.ai/install`, npm `opencode-ai`, scoop, choco, brew, pacman, AUR, mise, plus a desktop app (`packages/desktop`, same version 1.18.34). — [R: README.md](https://github.com/anomalyco/opencode/blob/dev/README.md)
- Fork/rename note (brief): the Go-based Charm "Crush" descends from the earlier opencode-ai/opencode Go project, not from this repo. It is covered elsewhere.

### Inferences
- The project is very high velocity with a large community. It is commercially backed: there is a `console` package (opencode.ai Zen/console), an `enterprise` package and a `slack` package, which suggests the company monetizes services around it. The open core appears to remain MIT.
- Releases ship several times a week, so any fork or deep embed has to absorb continuous upstream change.

### Gaps
- I did not get an exact contributor count (the GitHub page fetch didn't expose it). I also couldn't verify funding amounts. Business-model details (Zen/Go/Black plans) were not verified from primary sources.

## 2. Providers / models and subscription auth

### Takeaway
The production path still runs on the Vercel AI SDK v5/v6 (`ai` plus about 20 `@ai-sdk/*` packages), with a model catalog sourced from models.dev. A new in-house, Effect-based `@opencode-ai/llm` layer exists behind an opt-in flag (`OPENCODE_EXPERIMENTAL_NATIVE_LLM`). Subscription login is supported for ChatGPT/Codex and GitHub Copilot. Claude Pro/Max OAuth was dropped after Anthropic's 2026 enforcement.

### Cited Findings
- `@ai-sdk/*` dependencies include alibaba, amazon-bedrock, anthropic, azure, cerebras, cohere, deepinfra, gateway, google, google-vertex, groq, mistral, openai, openai-compatible, perplexity, togetherai, vercel and xai, plus `ai`. — [R: packages/opencode/package.json](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/package.json)
- `session/llm.ts` imports `streamText` / `wrapLanguageModel` from `ai`. Its comment reads "Runtime seam: native is an opt-in adapter over @opencode-ai/llm", gated by `flags.experimentalNativeLlm` (env `OPENCODE_EXPERIMENTAL_NATIVE_LLM`). — [R: packages/opencode/src/session/llm.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/session/llm.ts), [R: src/effect/runtime-flags.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/effect/runtime-flags.ts)
- `@opencode-ai/llm` describes itself as a "Schema-first LLM core … provider quirks live in adapters". It has a provider-neutral event stream across OpenAI Chat, OpenAI Responses, Anthropic Messages, Gemini, Bedrock Converse and OpenAI-compatible endpoints, and prompt caching is "on by default" with automatic breakpoints (last tool, last system, latest user message). — [R: packages/llm/README.md](https://github.com/anomalyco/opencode/blob/dev/packages/llm/README.md)
- The catalog comes from models.dev: there is `core/src/models-dev.ts`, and CONTEXT.md says "A shared ingestion adapter partitions legacy and models.dev AI-SDK-shaped options before routing." — [R: CONTEXT.md](https://github.com/anomalyco/opencode/blob/dev/CONTEXT.md)
- Built-in auth/provider plugins live in `src/plugin/`: `openai/codex.ts` (Codex/ChatGPT OAuth, with a `ws-pool.ts` websocket pool), `github-copilot/`, azure, cerebras, cloudflare, digitalocean, modal, snowflake-cortex and xai. I found no Anthropic OAuth plugin in the tree. — [R: packages/opencode/src/plugin](https://github.com/anomalyco/opencode/tree/dev/packages/opencode/src/plugin)
- `agent.ts` special-cases `providerID === "openai" && authInfo?.type === "oauth"` (the ChatGPT-subscription path). — [R: src/agent/agent.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/agent/agent.ts)
- Anthropic controversy: Anthropic began blocking third-party OAuth against Max plans on Jan 9 2026 and revised its Consumer Terms to forbid Free/Pro/Max OAuth tokens in third-party harnesses. OpenCode removed subscription-key support. — [letsdatascience](https://letsdatascience.com/news/anthropic-revises-terms-to-bar-third-party-harnesses-b7377cc7), [Gigazine 2026-02-20](https://gigazine.net/gsc_news/en/20260220-anthropic-third-party-block). A secondary source also claims "subscription coverage cut for OpenClaw / OpenCode / NanoClaw on April 4, 2026" (date unverified).
- Community plugins re-adding Claude subscription auth exist for other harnesses (for example pi-claude-code-auth). — [pi-claude-code-auth](https://github.com/cgaravitoq/pi-claude-code-auth/)

### Inferences
- For kitchen's multi-provider worker pool, opencode already covers essentially every API provider plus Copilot and ChatGPT subscriptions. It does not legitimately cover Claude Pro/Max subscriptions; that requires the official Claude Code/Agent SDK instead.
- The provider layer is in transition (AI SDK to native `@opencode-ai/llm`). `@opencode-ai/llm` could be borrowed standalone as a provider-neutral streaming client, but it is Effect-native.

### Gaps
- I did not confirm whether `@opencode-ai/llm` is published to npm as a stable package.

## 3. Architecture: client/server, API, SDK, events, storage, agent loop, compaction

### Takeaway
opencode is a true client/server system. A Bun HTTP server (`opencode serve`, default 127.0.0.1:4096, OpenAPI 3.1 at `/doc`) is driven by the TUI, desktop app, web UI, ACP adapter and SDKs. Since early 2026 it stores state in SQLite (Drizzle) with an event-sourced `event` table keyed by `(aggregate_id, seq)`. Durable per-session event replay is exposed as `sessions.events({sessionID, after})`. This is structurally very close to what kitchen plans.

### Cited Findings
- Server: `opencode serve --port 4096 --hostname 127.0.0.1 [--mdns] [--cors]`. HTTP basic auth comes from `OPENCODE_SERVER_PASSWORD` / `OPENCODE_SERVER_USERNAME`. The OpenAPI 3.1 spec is at `/doc`. There are two SSE endpoints, `GET /global/event` and `GET /event` (bus events, which start with `server.connected`). The TUI is a client of this server, and the `/tui/*` endpoints let external clients drive it. — [opencode.ai/docs/server](https://opencode.ai/docs/server/)
- Legacy API surface (in `packages/sdk/openapi.json`):
  - Sessions: `/session`, `/session/{id}/message`, `/session/{id}/prompt_async`, `/abort`, `/fork`, `/summarize`, `/revert`, `/children`, `/diff`, `/todo`, `/session/status`.
  - Approvals and questions: `/permission/{requestID}/reply`, `/question/{requestID}/reply`.
  - Integrations: `/mcp/*`, `/lsp`, `/pty/*` (with a WebSocket connect token), `/vcs/diff`.
  - Sync: `/sync/start|replay|steal|history`.
  - Experimental: `/experimental/worktree` (GET/POST/DELETE, reset), `/experimental/workspace` (+ `warp`, `sync-list`, `status`, `adapter`), `/experimental/session/{id}/background`, `/experimental/control-plane/move-session`.
  — [R: packages/sdk/openapi.json](https://github.com/anomalyco/opencode/blob/dev/packages/sdk/openapi.json)
- New "v2" API under `/api/*`:
  - Sessions: `/api/session`, `/api/session/active`, `/api/session/{id}/prompt|agent|model|compact|wait|interrupt|context|history|event|message`, `/api/session/{id}/revert/stage|clear|commit`.
  - Permissions: `/api/permission/request`, `/api/session/{id}/permission/{requestID}/reply`, `/api/permission/saved`.
  - Everything else: `/api/integration/*`, `/api/credential/*`, `/api/fs/*`.
  — [R: openapi.json](https://github.com/anomalyco/opencode/blob/dev/packages/sdk/openapi.json)
- SDKs: `@opencode-ai/sdk` 1.18.34 is the legacy Hey-API-generated client. `@opencode-ai/client` provides Promise and `/effect` clients generated from an "SDK Contract IR". `@opencode-ai/sdk-next` is an "Effect-native scoped OpenCode host for in-process applications" that "executes Server's assembled HTTP router in memory. It opens no listener" and exposes local-only `tools.register(...)`. sdk-next "will assume the existing `@opencode-ai/sdk` name after legacy consumers migrate." — [R: packages/sdk-next/README.md](https://github.com/anomalyco/opencode/blob/dev/packages/sdk-next/README.md), [R: CONTEXT.md](https://github.com/anomalyco/opencode/blob/dev/CONTEXT.md)
- Event semantics (CONTEXT.md):
  - `sessions.events({ sessionID, after })` "replays durable events after the optional aggregate sequence, continues with newly committed durable events, excludes live-only fragments", transported as SSE.
  - `events.subscribe()` is an instance-wide live stream with "no replay guarantee".
  - Neither stream auto-reconnects; "durable sequence-based resume remains explicit composition above the generated client."
  - No cross-instance aggregation stream exists yet.
  — [R: CONTEXT.md](https://github.com/anomalyco/opencode/blob/dev/CONTEXT.md)
- Event sourcing (`src/sync/README.md`): "basic event sourcing system for session replayability … one device to control and modify the session, and allow multiple other devices to 'sync'". Total order comes from "a simple sequence id". Events are emitted before mutation, and "projectors" apply the effects. Events are defined via `SyncEvent.define({type, version, aggregate, schema})`. — [R: src/sync/README.md](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/sync/README.md)
- Storage: SQLite via Drizzle at `<data>/opencode.db`, with Bun and Node drivers. Tables include `event_sequence(aggregate_id PK, seq, owner_id)` and `event(id, aggregate_id, seq, type, data json)` with a unique index on `(aggregate_id, seq)`, plus Session, Message, Part, Todo, Project, Workspace, Permission, Credential and Share tables. There are 38 migration files, the first dated 20260127 (the JSON-file storage was migrated to SQLite around late January 2026). — [R: core/src/event/sql.ts](https://github.com/anomalyco/opencode/blob/dev/packages/core/src/event/sql.ts), [R: core/src/database/database.ts](https://github.com/anomalyco/opencode/blob/dev/packages/core/src/database/database.ts), [R: src/storage/schema.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/storage/schema.ts)
- Agent loop (CONTEXT.md glossary):
  - Prompts are durably admitted to a session inbox ("Admitted Prompt") and promoted at a "Safe Provider-Turn Boundary".
  - A "Session Drain" runs provider turns until no continuation remains.
  - "Steering prompts" promote mid-drain, while queued prompts wait until idle.
  - `sessions.prompt(..., resume:false)` gives admit-only behavior.
  - Agent and model switches apply on the next provider turn.
  - System context is versioned in "Context Epochs" with a durable baseline, kept for provider prompt-cache stability, and changes are announced via "Mid-Conversation System Messages".
  - Oversized tool output is truncated, with the full text stored as a "Managed Tool Output File".
  — [R: CONTEXT.md](https://github.com/anomalyco/opencode/blob/dev/CONTEXT.md)
- Core loop files are `src/session/{prompt.ts, processor.ts, llm.ts, compaction.ts, overflow.ts, retry.ts, revert.ts, run-state.ts, status.ts, summary.ts}`. — [R: src/session](https://github.com/anomalyco/opencode/tree/dev/packages/opencode/src/session)
- Compaction: `compaction.ts` handles auto-compaction on overflow (`isOverflow` from `overflow.ts`) and an optional `compaction.prune` config that prunes old tool outputs above a minimum. A hidden `compaction` agent writes the summary. Plugins can hook `experimental.session.compacting` and `experimental.compaction.autocontinue`. Completed compaction starts a new Context Epoch. — [R: src/session/compaction.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/session/compaction.ts), [R: packages/plugin/src/index.ts](https://github.com/anomalyco/opencode/blob/dev/packages/plugin/src/index.ts)
- `sessions.active()` lists only foreground drains. "Background subagents and tasks do not make their parent Session active, and process restart clears the registry." — [R: CONTEXT.md](https://github.com/anomalyco/opencode/blob/dev/CONTEXT.md)

### Inferences
- opencode independently converged on kitchen's core persistence design: an append-only SQLite event log with per-aggregate seq and replay-after-seq over SSE. That validates the design and makes `core/src/event` plus `src/sync` prime borrow-from material.
- Execution state (drains, background jobs) is process-local and not durable across restarts, so a supervisor like kitchen's daemon still has to own crash recovery.
- Two API generations (legacy `/session/*` and new `/api/*`) coexist. Anything kitchen builds against should target the new `/api` + `@opencode-ai/client`, but that is explicitly "beta" ("must be settled before stabilization").

### Gaps
- I did not verify which WebSocket endpoints exist beyond the PTY connect and Codex ws pool; the event streams are SSE, not WebSocket.

## 4. Tools, permissions, sandboxing, LSP, MCP, ACP

### Takeaway
opencode has a standard coding toolset with glob-pattern permission rules (allow/ask/deny), first-class LSP, MCP (local and remote, with OAuth) and an ACP server (`opencode acp`). There is no OS-level sandbox: the default permission is `"*": "allow"`.

### Cited Findings
- Built-in tools (`src/tool/`): apply_patch, edit, write, read, glob, grep, shell, lsp, webfetch, websearch, mcp-websearch, task, todo/todowrite, question, skill, plan (plan-enter/exit), code-mode, external-directory. — [R: src/tool](https://github.com/anomalyco/opencode/tree/dev/packages/opencode/src/tool)
- Default permissions in `agent.ts`:
  ```
  "*": "allow"
  doom_loop: "ask"
  external_directory: { "*": "ask", <tmp, skill and reference dirs>: "allow" }
  question: "deny"
  plan_enter / plan_exit: "deny"
  read: { "*.env": "ask", "*.env.*": "ask", "*.env.example": "allow" }
  ```
  User config `permission` is layered on top. — [R: src/agent/agent.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/agent/agent.ts)
- Permission requests surface via the API (`GET /permission`, `POST /permission/{id}/reply`, new `/api/permission/request`, saved permissions). Plugins can intercept via the `permission.ask` hook. — [R: openapi.json](https://github.com/anomalyco/opencode/blob/dev/packages/sdk/openapi.json), [R: plugin/src/index.ts](https://github.com/anomalyco/opencode/blob/dev/packages/plugin/src/index.ts)
- `opencode run` has `--auto`, `--yolo` and `--dangerously-skip-permissions` flags. — [R: src/cli/cmd/run.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/cmd/run.ts)
- The only "sandbox" in the code is `@opencode-ai/codemode`, a JS code-mode tool sandbox for composing tools. It is not process isolation. A `packages/containers` package exists, but I did not inspect it. — [R: src/tool/code-mode.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/tool/code-mode.ts)
- ACP: `src/acp/` (agent, session, permission, tool, event, usage, config-option…) uses `@agentclientprotocol/sdk` 0.21.0. The command is `opencode acp --cwd`. v1.18.31 fixed ACP session model restore. — [R: src/acp](https://github.com/anomalyco/opencode/tree/dev/packages/opencode/src/acp), [releases](https://github.com/anomalyco/opencode/releases)
- MCP: API endpoints `/mcp`, `/mcp/{name}/connect|disconnect|auth|auth/callback|auth/authenticate` (MCP OAuth supported), plus a `src/mcp` module and `opencode mcp` CLI. LSP: `src/lsp`, the `lsp` tool, and `/lsp`, `/find/symbol` endpoints. Formatters: `/formatter`. — [R: openapi.json](https://github.com/anomalyco/opencode/blob/dev/packages/sdk/openapi.json)
- Security history: CVE-2026-22812 (CVSS 8.8) was an unauthenticated HTTP server enabling shell exec, fixed in 1.0.216 by adding server auth. CVE-2026-22813 (CVSS 9.4) was XSS in the web UI markdown renderer leading to RCE via WebSocket. — [lilting.ch writeup](https://lilting.ch/en/articles/opencode-cve-2026-22812-22813-rce). The same source's claims of "binding to 0.0.0.0" and "220,000 exposed instances" are unverified and look doubtful; current docs say the default hostname is 127.0.0.1. Further CVEs are listed at [vuln.today/tag/opencode](https://vuln.today/tag/opencode) (not reviewed).

### Inferences
- For kitchen workers, isolation must come from kitchen (worktree, plus container/user/namespace if desired). opencode's permission system gives a hook point to route "ask" events into kitchen's "what needs me" queue: subscribe to permission/question events and reply via the API.

## 5. Multi-agent, orchestration, worktrees

### Takeaway
opencode has primary and subagent modes, a `task` tool with resumable `task_id` and `background: true` async subagents, child sessions, and a git-worktree service. An experimental "workspace"/control-plane layer (`OPENCODE_EXPERIMENTAL_WORKSPACES`) supports local and remote targets and moving sessions between them. It has no user-facing orchestrator/manager agent with a dashboard of parallel heterogeneous workers; that is the gap kitchen fills.

### Cited Findings
- Agent schema `mode: "subagent" | "primary" | "all"`. Built-ins:
  - `build` (primary)
  - `plan` (primary, "Disallows all edit tools")
  - `general` (subagent)
  - `explore` (subagent, fast codebase search)
  - hidden `compaction`, `title` and `summary` agents
  The `default_agent` config cannot be a subagent. — [R: src/agent/agent.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/agent/agent.ts)
- `task` tool params: `subagent_type`, `prompt`/`description`, optional `task_id` (resume the same subagent session), optional `background` ("launches the subagent asynchronously and returns immediately… You will be notified when it completes. DO NOT sleep, poll"). Prompt text: "Launch multiple agents concurrently whenever possible". Background runs use `BackgroundJob` (`start/wait/extend/promote/cancel`, in `@opencode-ai/core/background-job`), which is instance-scoped. — [R: src/tool/task.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/tool/task.ts), [R: src/tool/task.txt](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/tool/task.txt), [R: src/background/job.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/background/job.ts)
- Subagents run as child sessions (`GET /session/{id}/children`). Per-agent `model` can differ ({providerID, modelID}), so a parent on model A can spawn subagents on provider B. — [R: agent.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/agent/agent.ts), [R: openapi.json](https://github.com/anomalyco/opencode/blob/dev/packages/sdk/openapi.json)
- Worktree service (`src/worktree/index.ts`, 623 lines): `git worktree add --no-checkout -b <branch>` under `<data>/worktree/<projectID>/`, with list/remove/reset, a start-command hook and name generation. It is exposed at `/experimental/worktree`. — [R: src/worktree/index.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/worktree/index.ts)
- Control plane: `WorkspaceAdapter {name, description, configure, create…}` with a `worktree` adapter, `Target = local{directory} | remote{url, headers}`, and endpoints `/experimental/workspace`, `/warp` and `/control-plane/move-session`. It is gated by `OPENCODE_EXPERIMENTAL_WORKSPACES`. — [R: src/control-plane/types.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/control-plane/types.ts), [R: src/effect/runtime-flags.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/effect/runtime-flags.ts)
- The ecosystem fills the orchestration gap externally. For example, "opentree" manages multiple agent sessions, each in a git worktree + tmux window, and later versions drive agents over ACP. — [pkg.go.dev opentree](https://pkg.go.dev/github.com/axelgar/opentree@v1.0.1); there is also an "oh-my-opencode-slim" worktrees skill — [skills.sh](https://www.skills.sh/alvinunreal/oh-my-opencode-slim/worktrees)

### Inferences
- opencode's multi-agent model is hierarchical delegation inside one session tree on one server instance. It has no cross-instance supervisor, no "needs attention" aggregation, and background jobs are not durable across restarts. Kitchen's Kitchen Manager plus a fleet of isolated, heterogeneous workers is a layer above this.
- The experimental workspace/remote-target and move-session features overlap with kitchen's direction and may evolve toward it, which is both a competitive and a churn signal.

## 6. Extension model

### Takeaway
opencode has a rich extension model: JS/TS plugins with hooks (`@opencode-ai/plugin`), custom agents (markdown or JSON), custom tools, commands, skills, themes and TUI plugins. These are configured via `opencode.json(c)` and `.opencode/` directories.

### Cited Findings
- Plugin hooks:
  - Chat: `chat.message`, `chat.params`, `chat.headers`
  - Permissions and commands: `permission.ask`, `command.execute.before`
  - Tools and shell: `tool.execute.before/after`, `tool.definition`, `shell.env`
  - Experimental: `experimental.chat.messages.transform`, `experimental.chat.system.transform`, `experimental.session.compacting`, `experimental.compaction.autocontinue`, `experimental.text.complete`
  - Event subscriptions
  The package includes `tool.ts`, `shell.ts`, `tui.ts`, `example-workspace.ts` (plugin-provided workspace adapters) and a `v2/` API. — [R: packages/plugin/src](https://github.com/anomalyco/opencode/tree/dev/packages/plugin/src)
- The repo's own `.opencode/` dir shows the layout: `agent/`, `command/`, `plugins/`, `skills/`, `themes/`, `tool/`, `opencode.jsonc`, `tui.json`. — [R: .opencode](https://github.com/anomalyco/opencode/tree/dev/.opencode)
- In-process tool registration is available via sdk-next `tools.register(...)`. — [R: sdk-next README](https://github.com/anomalyco/opencode/blob/dev/packages/sdk-next/README.md)
- Instructions come from `AGENTS.md` (global plus upward project discovery). Disable project config with `OPENCODE_DISABLE_PROJECT_CONFIG`. — [R: CONTEXT.md](https://github.com/anomalyco/opencode/blob/dev/CONTEXT.md)

### Inferences
- Kitchen could ship a "kitchen" opencode plugin per worker that injects a reporting tool (for example `report_status` / `request_human`), sets `shell.env`, and forwards permission asks. That is a low-cost integration.

## 7. Embeddability and fit for kitchen

### Takeaway
opencode is strong as a worker engine and as a borrow-from reference. It is weak as the orchestration base: its internals are Effect-heavy, mid-refactor, and organized around a single instance with one interactive user. Recommendation: **run opencode headless as one worker backend (via `opencode serve` / ACP / `@opencode-ai/client`), borrow its event-sourcing and context-epoch designs, and do not fork it as kitchen's core.**

### Cited Findings
- Headless options:
  - `opencode serve` (HTTP+SSE, basic auth)
  - `opencode run` with `--format`, `--attach <url>`, `--session`, `--continue`, `--fork`, `--model`, `--agent`, `--variant`, `--dir`, `--port`, `--replay`, `--auto`/`--yolo`
  - `opencode acp` (stdio ACP agent)
  - in-process `@opencode-ai/sdk-next` (`OpenCode.create()`, no listener, Effect Scope-managed)
  — [R: src/cli/cmd/run.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/cmd/run.ts), [R: src/cli/cmd/acp.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/cmd/acp.ts), [R: sdk-next README](https://github.com/anomalyco/opencode/blob/dev/packages/sdk-next/README.md)
- Per-session control the Kitchen Manager needs is available via API: create session (with optional `location`), prompt / prompt_async with `resume`, `switchAgent`, `switchModel`, `interrupt`, `wait`, `compact`, `revert stage/commit`, `diff`, durable `event` stream with `after` seq, and permission/question reply. — [R: CONTEXT.md](https://github.com/anomalyco/opencode/blob/dev/CONTEXT.md), [R: openapi.json](https://github.com/anomalyco/opencode/blob/dev/packages/sdk/openapi.json)
- v1.18.34 adds "namespaced session and parent-session identity headers" on model requests, which helps attribute cost per worker. — [releases](https://github.com/anomalyco/opencode/releases)
- The beta client API is explicitly unstable ("whether the stable Session namespace should instead be singular `session` must be settled before stabilization"). The legacy `@opencode-ai/sdk` will be replaced by sdk-next. — [R: CONTEXT.md](https://github.com/anomalyco/opencode/blob/dev/CONTEXT.md)

### Inferences
- **Build-on (worker engine): good fit.** Run one `opencode serve` per worktree (or per project with `location`), on a random port with `OPENCODE_SERVER_PASSWORD`, then:
  - subscribe to `/api/session/{id}/event?after=seq`
  - map permission/question requests to kitchen's "needs me" queue
  - persist opencode seq cursors in kitchen's own log

  ACP is the more provider-agnostic alternative, since the same adapter also covers Claude Code, Gemini CLI and others, at the cost of less fidelity.
- **Use as orchestration base: poor fit.**
  - It has no supervisor across instances, and background state isn't durable.
  - The TUI and data model assume a single interactive user.
  - Its pervasive Effect-TS idiom imposes a framework choice.
  - There are two API generations plus SDK churn, and several releases a week.
  - Its Claude subscription path is gone, so Claude workers would need Claude Code/Agent SDK anyway, which argues for kitchen owning orchestration and treating opencode as one of several engines.
- **Borrow-from:**
  - the `event` / `event_sequence` schema and projector pattern (`src/sync`)
  - SSE replay-after-seq semantics with explicit (non-automatic) resume
  - Context Epoch / mid-conversation system-message design for prompt-cache stability
  - managed tool-output truncation
  - permission rule evaluation (`src/permission/evaluate.ts`, `arity.ts`)
  - worktree service flow
  - `@opencode-ai/llm` caching-breakpoint policy
- MIT licensing makes copying code legally simple, with attribution.

### Gaps
- I didn't measure the per-instance resource footprint (RAM per `opencode serve`), which matters for running N workers. I didn't test whether one server can host many worktree directories concurrently via `location` / `x-opencode-directory`; the legacy API supports per-directory instances, but I did not verify this.

## 8. Weaknesses, known issues, churn risk

### Takeaway
The main risks are extreme churn (architecture rewrite in progress), Effect-TS lock-in, a history of serious security bugs in the server/web UI, a lost Claude subscription path, and a sprawling commercial monorepo.

### Cited Findings
- Active migrations visible in the tree:
  - legacy `/session` to `/api` endpoints
  - `@opencode-ai/sdk` to `sdk-next` / `@opencode-ai/client`
  - AI SDK to native `@opencode-ai/llm` (flagged)
  - Bus events to SyncEvent ("This works, but you shouldn't publish sync event like this (should fail in the future)")
  - `core` package extraction ("Keeps the legacy service instance-scoped while sharing the core registry engine")
  — [R: src/sync/README.md](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/sync/README.md), [R: src/background/job.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/background/job.ts), [R: session/llm.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/session/llm.ts)
- Many features remain experimental and env-flagged (workspaces, native LLM, filewatcher, references). — [R: src/effect/runtime-flags.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/effect/runtime-flags.ts)
- CVE-2026-22812 and CVE-2026-22813 (server auth and web UI XSS to RCE), with later CVEs listed in aggregators. — [lilting.ch](https://lilting.ch/en/articles/opencode-cve-2026-22812-22813-rce), [vuln.today](https://vuln.today/tag/opencode)
- Anthropic's subscription-OAuth ban removed the Claude Pro/Max path. — [Gigazine](https://gigazine.net/gsc_news/en/20260220-anthropic-third-party-block)

### Inferences
- Pinning a version and talking only over a narrow protocol boundary (HTTP/SSE or ACP) contains the churn. Linking internals (sdk-next in-process, plugins using `experimental.*` hooks) exposes kitchen to frequent breakage.

### Gaps
- I didn't survey the GitHub issue tracker for systemic bugs (memory or performance, session corruption); there were no reliable sources in this pass. I found no 2026 HN thread with substantive criticism in this pass.
