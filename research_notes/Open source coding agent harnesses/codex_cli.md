# OpenAI Codex CLI (openai/codex, codex-rs) as a worker engine for kitchen

Method note: primary evidence comes from a shallow clone of `openai/codex` at commit `b741e480` (committed 2026-10-03 07:45 UTC), read directly. Citations to "repo:" paths point at that tree on GitHub (`https://github.com/openai/codex/blob/main/<path>`). The GitHub REST API was blocked from this environment, so star, contributor and release counts come from the rendered releases page and secondary sources.

## 1. License, language, repo layout, size, cadence, and what is open vs proprietary

### Takeaway
Codex is Apache-2.0, written mostly in Rust (`codex-rs`, about 118 crates), very large (about 128k stars, 20k forks), and moves extremely fast: several alpha tags a day and a stable release every few days. The CLI, TUI, app-server, protocols, SDKs and sandbox are open. Cloud tasks, the ChatGPT backend, model catalog and hosted features are services you reach through open client crates.

### Cited Findings
- The license is Apache License 2.0 — [repo: LICENSE](https://github.com/openai/codex/blob/main/LICENSE). There is also a CLA (`docs/CLA.md`) — [repo docs](https://github.com/openai/codex/tree/main/docs)
- Top-level layout: `codex-rs/` (Rust workspace), `codex-cli/` (npm wrapper `@openai/codex`), `sdk/typescript`, `sdk/python`, `sdk/python-runtime`, `docs/`. The build uses both Cargo and Bazel (`MODULE.bazel`, `BUILD.bazel`), plus a Nix flake — [repo root](https://github.com/openai/codex)
- Notable `codex-rs` crates (about 118 directories): `core`, `core-api`, `protocol`, `exec`, `exec-server`, `exec-server-protocol`, `tui`, `cli`, `app-server`, `app-server-protocol`, `app-server-client`, `app-server-daemon`, `app-server-transport`, `apply-patch`, `execpolicy`, `linux-sandbox`, `bwrap`, `windows-sandbox-rs`, `windows-sandbox-service`, `sandboxing`, `process-hardening`, `network-proxy`, `rollout`, `rollout-trace`, `thread-store`, `state`, `hooks`, `skills`, `plugin`, `core-plugins`, `codex-mcp`, `rmcp-client`, `model-provider`, `model-provider-info`, `models-manager`, `ollama`, `lmstudio`, `aws-auth`, `login`, `chatgpt`, `cloud-tasks`, `cloud-tasks-client`, `backend-client`, `agent-graph-store`, `agent-roles`, `agent-message-board-client`, `worktree`, `memories`, `code-mode*`, `realtime-webrtc`, `responses-api-proxy` — [repo: codex-rs/](https://github.com/openai/codex/tree/main/codex-rs)
- In-repo versions are placeholders (`version = "0.0.0"` in the workspace Cargo.toml, and `0.0.0-dev` in the npm `package.json` files). Versions are stamped at release time, and the crates are not maintained as published, semver-versioned library crates — [repo: codex-rs/Cargo.toml](https://github.com/openai/codex/blob/main/codex-rs/Cargo.toml)
- The releases page on 2026-10-03 showed latest stable **0.160.0** (Oct 1, 2026) and nine `0.162.0-alpha.N` tags between Oct 2 and Oct 3. It also showed about **128k stars and 20k forks** — [GitHub releases](https://github.com/openai/codex/releases)
- Earlier secondary counts: 103,504 stars by July 19, 2026, 840+ total releases, and 400+ contributors (date of that figure unclear). Q1 2026 had 37 releases — [gradually.ai stats](https://www.gradually.ai/en/codex-statistics/); [havoptic Q1 2026](https://www.havoptic.com/blog/deep-dive-openai-codex-q1-2026)
- `codex-app-server-daemon` "backs the machine-readable `codex app-server` lifecycle commands used by remote clients such as the desktop and mobile apps". This means the desktop and mobile apps are clients of the open app-server, while the apps themselves are not in this repo — [repo: codex-rs/app-server-daemon/README.md](https://github.com/openai/codex/blob/main/codex-rs/app-server-daemon/README.md)
- `thread-store` defines a `ThreadStore` trait with local and in-memory implementations; "Other storage implementations may live outside this repository". This points to proprietary or cloud stores — [repo: codex-rs/thread-store/README.md](https://github.com/openai/codex/blob/main/codex-rs/thread-store/README.md)

### Inferences
- The engine and the client protocol are fully open. The closed parts are services: the ChatGPT backend, Codex Cloud task execution, remote plugins and marketplace, hosted Apps MCP, and the desktop app UI. The open repo still contains client code for all of these, which is part of why it is so large.
- With alpha tags arriving several times a day, any fork or deep library embedding faces constant churn.

### Gaps
- I could not get exact current contributor counts or a commit-frequency figure because the GitHub API was blocked.

## 2. Embedding surfaces: `codex exec --json`, `codex app-server`, `codex mcp-server`, SDKs, Rust library use

### Takeaway
The app-server is the real embedding surface. It speaks JSON-RPC (without the `"jsonrpc"` header) over stdio, a unix socket or websocket, uses a Thread/Turn/Item model, and sends approvals as server-to-client requests. Its TS and JSON schemas can be generated per installed version, and unstable parts sit behind an `experimentalApi` opt-in. The TypeScript SDK is a thin wrapper over `codex exec --experimental-json`. The Python SDK drives threads and turns. Using `codex-core` directly as a Rust library is possible but unsupported.

### Cited Findings
- **App-server transports.** `--listen` accepts `stdio://` (the default), `unix://`, `unix://PATH`, `ws://IP:PORT`, or `off`. `codex app-server --stdio` is the same as `--listen stdio://` — [repo: codex-rs/app-server/src/main.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server/src/main.rs); [repo: codex-rs/cli/src/main.rs](https://github.com/openai/codex/blob/main/codex-rs/cli/src/main.rs)
- The official docs say "WebSocket transport is experimental and unsupported". Websocket auth flags are `--ws-auth signed-bearer-token` and `--ws-auth capability-token` — [Codex docs: App Server](https://learn.chatgpt.com/docs/app-server) (developers.openai.com/codex/app-server now 308-redirects here)
- **Protocol shape.** It is JSON-RPC 2.0 with the `"jsonrpc":"2.0"` field omitted on the wire. The client must send one `initialize` request first. The capability `experimentalApi` gates unstable methods and fields: "If a client sends an experimental method or field without opting in, app-server rejects it." The core primitives are Thread, Turn and Item — [Codex docs: App Server](https://learn.chatgpt.com/docs/app-server)
- **Schema generation.** `codex app-server generate-ts --out DIR` and `codex app-server generate-json-schema --out DIR` emit schemas for the installed version — [Codex docs: App Server](https://learn.chatgpt.com/docs/app-server). The repo also checks in `app-server-protocol/schema/{json,typescript}` — [repo: codex-rs/app-server-protocol](https://github.com/openai/codex/tree/main/codex-rs/app-server-protocol)
- Fields are marked experimental per field in Rust, e.g. `#[experimental("thread/start.dynamicTools")]`, and the error text reads "{reason} requires experimentalApi capability" — [repo: app-server-protocol/src/experimental_api.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/experimental_api.rs). The v2 protocol lives in `protocol/v2/`, and `docs/protocol_v1.md` documents the legacy v1 protocol — [repo: codex-rs/docs](https://github.com/openai/codex/tree/main/codex-rs/docs)
- **Method and notification families** (taken from `common.rs` at HEAD):
  - Threads: `thread/start|resume|fork|read|list|loaded/list|archive|unarchive|delete|rollback|revert|compact/start|name/set|metadata/update|settings/update|shellCommand|items/list|turns/list|search|goal/set|get|clear|queue/add|list|start|…|backgroundTerminals/list|terminate|unsubscribe`
  - Turns: `turn/start|steer|interrupt|settings/update`
  - Notifications: `thread/started`, `thread/status/changed`, `thread/tokenUsage/updated`, `thread/compacted`, `turn/started|completed`, `turn/diff/updated`, `turn/plan/updated`, `item/started|completed`, `item/agentMessage/delta`, `item/reasoning/*`, `item/commandExecution/outputDelta`, `item/fileChange/outputDelta|patchUpdated`, `item/mcpToolCall/progress`, `hook/started|completed`
  - Server-to-client requests (approvals and user input): `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, `item/permissions/requestApproval`, `item/tool/requestUserInput`, `item/tool/call` (client-defined dynamic tools), `mcpServer/elicitation/request`. `serverRequest/resolved` is also sent.
  - Other: `review/start`, `model/list`, `config/read|value/write|batchWrite`, `command/exec*`, `process/spawn|kill|writeStdin`, `fs/*` (read, write, watch), `mcpServer/*`, `skills/*`, `plugin/*`, `hooks/list`, `account/login/start|logout|read|rateLimits/read`, `account/chatgptAuthTokens/refresh`, `rollout/compress`, `project/*`, `remoteControl/*`, `thread/realtime/*`

  Source: [repo: app-server-protocol/src/protocol/common.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/protocol/common.rs)
- **`thread/start` params.** Stable fields: `model`, `modelProvider`, `cwd`, `approvalPolicy`, `approvalsReviewer`, `sandbox`, `config` (a map of overrides), `baseInstructions`, `developerInstructions`, `personality`, `ephemeral`, `serviceName`. Experimental fields: `dynamicTools`, `permissions`, `multiAgentMode`, `historyMode`, `environments`, `runtimeWorkspaceRoots`, `allowProviderModelFallback`, `projectId`, `experimentalRawEvents` — [repo: app-server-protocol/src/protocol/v2/thread.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/protocol/v2/thread.rs)
- **`codex exec --json`.** Emits JSONL `ThreadEvent`s: `thread.started`, `turn.started`, `turn.completed` (with usage), `turn.failed`, `item.started`, `item.updated`, `item.completed`, `error` — [repo: codex-rs/exec/src/exec_events.rs](https://github.com/openai/codex/blob/main/codex-rs/exec/src/exec_events.rs)
- **TypeScript SDK `@openai/codex-sdk`.** "wraps the `codex` CLI … spawns the CLI and exchanges JSONL events over stdin/stdout". Internally it runs `codex exec --experimental-json`. The API is `new Codex()`, `startThread({workingDirectory, skipGitRepoCheck})`, `thread.run()` / `runStreamed()`, `outputSchema` for structured output, and `resumeThread(id)`; threads persist in `~/.codex/sessions`. It requires Node 18+ — [repo: sdk/typescript/README.md](https://github.com/openai/codex/blob/main/sdk/typescript/README.md); [repo: sdk/typescript/src/exec.ts](https://github.com/openai/codex/blob/main/sdk/typescript/src/exec.ts)
- **Python SDK.** `pip install openai-codex` gives `from openai_codex import Codex`, `codex.thread_start()` and `thread.run()` returning a TurnResult — [repo: sdk/python/README.md](https://github.com/openai/codex/blob/main/sdk/python/README.md)
- **Other surfaces.** `codex exec-server` is a separate small JSON-RPC server for spawning and controlling PTY subprocesses, with an `ExecServerClient` in Rust — [repo: codex-rs/exec-server/README.md](https://github.com/openai/codex/blob/main/codex-rs/exec-server/README.md). Example crates include `thread-manager-sample` and `app-server-client`; the app-server daemon is marked "experimental and its lifecycle contract may change" — [repo: app-server-daemon/README.md](https://github.com/openai/codex/blob/main/codex-rs/app-server-daemon/README.md)

### Inferences
- For kitchen, a Bun daemon that spawns one long-lived `codex app-server` over stdio, or one per worker for isolation, maps cleanly onto kitchen's own model:
  - `thread/*`, `turn/*` and `item/*` notifications can be normalized into kitchen's append-only SQLite event log.
  - Approval requests (`item/*/requestApproval`, `requestUserInput`) feed directly into the "what needs me" view.
  - TS types come from `generate-ts`.
- The TS SDK is not suitable for kitchen. It goes through `exec`, which is non-interactive (no approval round-trips) and starts one process per run. Use the app-server protocol directly.
- Stability: there is no semver or LTS guarantee for the protocol. Stability is managed field by field through the `experimentalApi` gate. Version-pin the codex binary and regenerate schemas when upgrading.
- `codex mcp-server` was not examined at HEAD (see Gaps). The app-server has superseded it as the rich interface.

### Gaps
- I did not verify whether `codex mcp-server` still exists or is deprecated at HEAD (I did not grep the `cli` subcommands for it).
- I found no published compatibility or deprecation policy document for app-server v2 beyond the experimental gating.

## 3. Provider support and non-OpenAI models

### Takeaway
Codex only speaks the OpenAI **Responses API**. `wire_api = "chat"` was removed as a hard error from Feb 1, 2026. Built-in providers are `openai`, `amazon-bedrock` / `amazon-bedrock-runtime`, `ollama` and `lmstudio`. Claude, Gemini and similar models work only through a Responses-compatible gateway (LiteLLM, Bifrost, OpenRouter-style proxies), and results depend on the model. This is the single biggest mismatch with kitchen's multi-provider goal.

### Cited Findings
- `enum WireApi { Responses }` is the only variant. Deserializing `"chat"` fails with "`wire_api = \"chat\"` is no longer supported. How to fix: set `wire_api = \"responses\"`". `ollama-chat` was also removed — [repo: model-provider-info/src/lib.rs](https://github.com/openai/codex/blob/main/codex-rs/model-provider-info/src/lib.rs)
- **Deprecation timeline.** Announced Dec 9, 2025, with full removal on Feb 1, 2026. Codex's stated reason: chat/completions "has increasingly hampered our ability to improve Codex". Commenters on the discussion said LiteLLM, Ollama, LM Studio and other gateways often lacked good `/responses` support — [GitHub discussion #7782](https://github.com/openai/codex/discussions/7782)
- **Built-in providers.** `built_in_model_providers()` registers `openai`, `amazon-bedrock`, `amazon-bedrock-runtime`, `ollama` and `lmstudio`, with the OSS providers created through `create_oss_provider(port, WireApi::Responses)`. There is no native Anthropic or Gemini adapter in the model-provider crates — [repo: model-provider-info/src/lib.rs](https://github.com/openai/codex/blob/main/codex-rs/model-provider-info/src/lib.rs)
- **`[model_providers.<id>]` keys:** `name`, `base_url`, `env_key`, `env_key_instructions`, `experimental_bearer_token`, `auth` (command-backed token), `gateway_oauth`, `aws` (SigV4), `wire_api`, `capabilities`, `query_params`, `http_headers`, `env_http_headers`, `request_max_retries`, `stream_max_retries`, `stream_idle_timeout_ms`, `websocket_connect_timeout_ms`, `requires_openai_auth`, `supports_websockets` — [repo: model-provider-info/src/lib.rs](https://github.com/openai/codex/blob/main/codex-rs/model-provider-info/src/lib.rs)
- **Per-thread provider selection.** `thread/start` accepts `modelProvider` and `model`, so a single app-server can run threads on different configured providers — [repo: v2/thread.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/protocol/v2/thread.rs)
- **Practice in Sept 2026.** A guide calls direct TOML "native `/responses` support". It says Chat-Completions-only providers (DeepSeek, Claude, Kimi, local Ollama) need a translating proxy such as "CC Switch", and warns that "lightweight models … often fail to properly serialize tool outputs, causing loop crashes". It also says "Starting with version 0.134.0, Codex completely dropped support for inline `[profiles.x]` blocks", which now live in `~/.codex/<profile>.config.toml` — [heyuan110 (2026-09-15)](https://www.heyuan110.com/posts/ai/2026-09-15-configuring-third-party-models-in-codex/) (secondary source; I did not verify the profile change in the repo)
- Gateways such as Bifrost translate Responses requests to Anthropic and Google, e.g. `codex --model anthropic/claude-sonnet-4-5-…`. They note that non-OpenAI models "must support tool use" — [Maxim/Bifrost article](https://www.getmaxim.ai/articles/openai-codex-cli-beyond-gpt-switch-between-claude-gemini-llama-and-more-using-bifrost/); [LLM Gateway docs](https://docs.llmgateway.io/guides/codex-cli)
- **GPT-tuned tools.** The core tool handlers include `apply_patch` (Codex's own patch grammar, `apply-patch` crate), a shell / `unified_exec` PTY tool (feature `unified_exec`, Stable), `plan`, `request_user_input`, `tool_search`, `multi_agents`, and MCP. The `apply_patch_freeform` feature (a custom/freeform-grammar tool) is now marked Removed — [repo: core/src/tools/handlers](https://github.com/openai/codex/tree/main/codex-rs/core/src/tools/handlers); [repo: features/src/lib.rs](https://github.com/openai/codex/blob/main/codex-rs/features/src/lib.rs)

### Inferences
- To run Claude or Gemini workers through Codex, kitchen would have to run or embed a Responses-API translation gateway. It would also inherit a GPT-tuned system prompt and the `apply_patch` envelope format, which non-OpenAI models follow less reliably. Responses-specific features such as encrypted reasoning items, server-side compaction and web search would degrade or vanish.
- Codex is a strong engine for OpenAI-model workers, and possibly local gpt-oss via Ollama or LM Studio. It is a weak universal engine. A better plan for kitchen: use the Codex app-server as one worker "adapter" for OpenAI or ChatGPT-plan jobs, and use the Claude Agent SDK, Gemini CLI or a paved-road library for other providers, all behind kitchen's own worker-adapter interface.

### Gaps
- I found no rigorous 2026 benchmark of Claude or Gemini quality inside the Codex harness versus their native harnesses. The evidence is anecdotal.

## 4. Agent loop, sandboxing, approvals, compaction, persistence and resume

### Takeaway
Codex has a mature single-agent loop:
- **Sandboxing:** OS sandboxes (Seatbelt on macOS, bubblewrap on Linux, a dedicated Windows sandbox service), with `sandbox_mode` and `approval_policy` presets.
- **Approvals:** an optional AI "Guardian" auto-reviewer.
- **Compaction:** auto-compaction by token limit.
- **Persistence and resume:** durable JSONL rollouts under `~/.codex/sessions`, with resume, fork, rollback and revert supported.

### Cited Findings
- `sandbox_mode` values are `read-only`, `workspace-write` and `danger-full-access`. `approval_policy` values include `untrusted`, `on-failure`, `on-request`, `never` and `granular` — [repo: protocol/src](https://github.com/openai/codex/tree/main/codex-rs/protocol/src)
- Linux sandboxing now uses **bubblewrap**. Codex prefers the system `bwrap` on PATH and falls back to a bundled `codex-resources/bwrap`. WSL1 is unsupported. "Legacy `SandboxPolicy` / `sandbox_mode` configs remain supported" — [repo: linux-sandbox/README.md](https://github.com/openai/codex/blob/main/codex-rs/linux-sandbox/README.md). Windows has its own `windows-sandbox-rs` and `windows-sandbox-service` crates and `windowsSandbox/*` RPCs; there are also `network-proxy` and `execpolicy` crates — [repo: codex-rs/](https://github.com/openai/codex/tree/main/codex-rs)
- **Guardian auto-review.** Feature `guardian_approval` is Stable. Setting `auto_review.circuit_break_action = "strict"` surfaces `tooManyDenials` — [repo: app-server/README.md](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md); [repo: features/src/lib.rs](https://github.com/openai/codex/blob/main/codex-rs/features/src/lib.rs)
- **Compaction.** Config keys are `model_auto_compact_token_limit` and `model_auto_compact_token_limit_scope`; RPCs are `thread/compact/start` and the `thread/compacted` notification — [repo: core/src/config/mod.rs](https://github.com/openai/codex/blob/main/codex-rs/core/src/config/mod.rs)
- **Rollout files** are stored at `~/.codex/sessions/YYYY/MM/DD/rollout-YYYY-MM-DDThh-mm-ss-<uuid>.jsonl`. The `rollout/compress` RPC and "Local rollout compression" now exist — [repo: rollout/src/list.rs](https://github.com/openai/codex/blob/main/codex-rs/rollout/src/list.rs)
- **Resume and branching RPCs:** `thread/resume`, `thread/fork`, `thread/rollback`, `thread/revert`, `thread/items/list`, `thread/turns/list`. Mid-turn steering uses `turn/steer` — [repo: common.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/protocol/common.rs)
- **Other Stable features:** `memories`, `goals` (thread goals), `hooks`, `plugins`, `skill_search`, `realtime_conversation` — [repo: features/src/lib.rs](https://github.com/openai/codex/blob/main/codex-rs/features/src/lib.rs)

### Inferences
- The sandbox and approval stack alone would take months to rebuild. Kitchen can use it per worker by running each worker thread with `cwd` set to its worktree, `sandbox: workspace-write`, and approvals relayed to the TUI.
- Codex keeps its own rollout JSONL, so kitchen would have a dual source of truth. Kitchen's SQLite log should treat Codex's thread id plus its rollout as the authoritative transcript and store normalized events, or use `ephemeral` threads plus kitchen's own log.

### Gaps
- I did not inspect the macOS Seatbelt profile details or seccomp usage at HEAD. The Linux path has moved from Landlock/seccomp to bwrap-based isolation, so older descriptions are outdated.

## 5. Multi-agent, parallel threads, worktrees, cloud tasks

### Takeaway
Codex now ships Stable built-in multi-agent support:
- **Tools:** `spawn_agent`, `send_input`, `wait_agent`, `list_agents`, `interrupt_agent`, `resume_agent`, `close_agent`, with agent roles and a parent/child graph store.
- **Worktrees:** managed git worktrees (`worktrees`, Stable).
- **Agent command center:** a TUI "agent command center".
- **Limits:** subagents inherit the parent's provider. The defaults are 6 threads and depth 1.

### Cited Findings
- Feature flags: `multi_agent` Stable, `multi_agent_v2` Stable, `multi_agent_v2_dynamic_tools` UnderDevelopment, `agent_message_board` UnderDevelopment, `worktrees` Stable ("Enable managed worktree creation and repository-aware sessions") — [repo: features/src/lib.rs](https://github.com/openai/codex/blob/main/codex-rs/features/src/lib.rs)
- `SpawnAgentArgs { message, items, agent_type, model, reasoning_effort, fork_context }` returns `{agent_id, nickname}` — [repo: core/src/tools/handlers/multi_agents/spawn.rs](https://github.com/openai/codex/blob/main/codex-rs/core/src/tools/handlers/multi_agents/spawn.rs)
- Child config copies the parent's provider: `config.model_provider = turn.provider.info().clone()` — [repo: core/src/agent/child_config.rs](https://github.com/openai/codex/blob/main/codex-rs/core/src/agent/child_config.rs)
- `DEFAULT_AGENT_MAX_THREADS = Some(6)`, `DEFAULT_AGENT_MAX_DEPTH = 1`, set through `agents.max_concurrent_threads_per_session`. There is also a "Shared token budget for the root thread and its sub-agents" — [repo: core/src/config/mod.rs](https://github.com/openai/codex/blob/main/codex-rs/core/src/config/mod.rs)
- `agent-graph-store` is "Storage-neutral parent/child topology for thread-spawned agents". The `agent-roles` crate loads role files — [repo: codex-rs/agent-graph-store](https://github.com/openai/codex/tree/main/codex-rs/agent-graph-store)
- The `worktree` crate exposes `ManagedWorktree { root, cwd, source_root, source_cwd, head_sha, branch }`, described as "A Desktop-compatible checkout and the cwd that should be used to start its thread", with `WorktreeSettings` and `DEFAULT_WORKTREE_KEEP_COUNT` — [repo: worktree/src/lib.rs](https://github.com/openai/codex/blob/main/codex-rs/worktree/src/lib.rs)
- The 0.160.0 release notes mention "Browse older tasks in the agent command center" — [GitHub releases](https://github.com/openai/codex/releases)
- Cloud: the `cloud-tasks`, `cloud-tasks-client` and `cloud-config` crates are clients for Codex Cloud (a proprietary backend) — [repo: codex-rs/](https://github.com/openai/codex/tree/main/codex-rs)
- One app-server hosts many threads (`thread/loaded/list`, `thread/unsubscribe`, `thread/status/changed`) — [repo: common.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/protocol/common.rs)

### Inferences
- Codex's own orchestrator-to-subagent model overlaps with kitchen's Kitchen Manager concept, but it is single-provider and in-process. Kitchen's differentiator is cross-provider orchestration with a durable external event log, so kitchen should own orchestration itself and use Codex only as a worker. Keep Codex's internal `multi_agent` turned off or limited for workers so you don't get nested orchestration.
- The desktop app's worktree convention (`worktree` crate) could be reused or mirrored so that kitchen worktrees stay compatible with Codex Desktop.

### Gaps
- I did not determine whether `thread/start.multiAgentMode` lets an external client fully disable subagent tools, nor the exact config key for that.

## 6. Extension: MCP, AGENTS.md, skills, hooks, plugins, prompts

### Takeaway
Codex has a full extension surface:
- **MCP:** an MCP client using `rmcp` with OAuth.
- **AGENTS.md:** layered instruction files.
- **Skills:** skills plus skill search.
- **Hooks:** Claude-style lifecycle hooks (`hooks.json`), Stable.
- **Plugins:** plugins and a marketplace.
- **Dynamic tools:** client-defined tools over the app-server.

### Cited Findings
- Hooks: "Enable Claude-style lifecycle hooks loaded from hooks.json files". The events are `PreToolUse`, `PostToolUse`, `PermissionRequest`, `PreCompact`, `SessionStart`, `Stop`, `SubagentStop` and `UserPromptSubmit`. Admins can enforce `allow_managed_hooks_only` in `requirements.toml` — [repo: hooks/src](https://github.com/openai/codex/tree/main/codex-rs/hooks/src); [repo: docs/config.md](https://github.com/openai/codex/blob/main/docs/config.md)
- Docs cover `agents_md.md`, `skills.md`, `slash_commands.md`, `execpolicy.md` and `exec.md` — [repo: docs/](https://github.com/openai/codex/tree/main/docs)
- Client-defined tools: `thread/start.dynamicTools` (experimental), called back through the `item/tool/call` server request — [repo: v2/thread.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/protocol/v2/thread.rs)
- MCP RPCs: `mcpServer/tool/call`, `mcpServer/oauth/login`, `config/mcpServer/reload`, `mcpServerStatus/list` — [repo: common.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/protocol/common.rs)
- `externalAgentConfig/detect|import` RPCs and the `external-agent-migration` crate import other agents' configs — [repo: common.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/protocol/common.rs)

### Inferences
- `dynamicTools` is the clean way for kitchen to inject tools into a Codex worker, e.g. `report_status`, `ask_manager` and `mark_blocked`, without forking. It is experimental, so pin the version.

### Gaps
- I did not verify the custom prompts (`~/.codex/prompts`) status at HEAD.

## 7. Forking risk and notable forks

### Takeaway
A fork means tracking a 118-crate Rust workspace that ships multiple alphas a day, so maintenance cost is very high. Community forks exist (Every Code adds multi-provider orchestration), but embedding the unmodified binary through the app-server is far lower risk.

### Cited Findings
- Release velocity: nine alpha tags in about 32 hours (0.162.0-alpha.2 through alpha.10, Oct 2–3, 2026) — [GitHub releases](https://github.com/openai/codex/releases)
- **Every Code (`just-every/code`, npm `@just-every/code`)** is a "community-driven fork of openai/codex". It adds Auto Drive multi-agent orchestration (`/plan`, `/code`, `/solve`), Auto Review and CDP browser integration, and orchestrates OpenAI, Claude, Gemini, Grok, DeepSeek and Qwen — [aicoolies: Every Code](https://aicoolies.com/tools/every-code); [gittrend: just-every/code](https://gittrend.io/repo/just-every/code)
- Other forks and wrappers include `NormalSubgroup/codex-fork`, `open-hax/codex`, `Lqm1/opencodex` and `cpiprint/open-codex`. Their activity is unverified — [search results](https://github.com/NormalSubgroup/codex-fork)
- Upstream's own breaking changes show the churn: chat wire API removed (Feb 2026) and inline profiles removed (0.134.0) — [discussion #7782](https://github.com/openai/codex/discussions/7782); [heyuan110](https://www.heyuan110.com/posts/ai/2026-09-15-configuring-third-party-models-in-codex/)

### Inferences
- A fork is not recommended for kitchen. Embedding a pinned `codex` binary through the app-server, plus a Responses gateway where needed, avoids rebase pain.

### Gaps
- I did not check the current activity or star counts of Every Code or the other forks.

## 8. Auth: ChatGPT plan vs API key, and third-party harness use

### Takeaway
Codex supports ChatGPT login (subscription billing) and API keys. The app-server even supports externally managed ChatGPT tokens. As of mid-2026, OpenAI publicly tolerates and endorses ChatGPT-plan use from third-party harnesses (OpenCode, Pi), unlike Anthropic. Building kitchen on the official Codex binary is the cleanest way to use ChatGPT-plan auth.

### Cited Findings
- Auth RPCs: `account/login/start|cancel|completed`, `account/logout`, `account/read`, `account/rateLimits/read|updated`, `account/usage/read`, `account/chatgptAuthTokens/refresh` (externally supplied ChatGPT tokens), plus Bedrock and gateway OAuth flows. The provider flag `requires_openai_auth` selects "OpenAI API key or ChatGPT login token" — [repo: common.rs](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/src/protocol/common.rs); [repo: model-provider-info/src/lib.rs](https://github.com/openai/codex/blob/main/codex-rs/model-provider-info/src/lib.rs)
- The TS SDK injects `CODEX_API_KEY` and reuses existing CLI auth — [repo: sdk/typescript/README.md](https://github.com/openai/codex/blob/main/sdk/typescript/README.md)
- July 2026: an OpenAI executive (Tibo Sottiaux) said ChatGPT accounts work with third-party harnesses and that Pi and OpenCode make up about 10% of Codex traffic. The "Codex for Open Source" program says developers "should code in the tools they prefer". The ToS "nothing … explicitly permits or prohibits" third-party use. Anthropic took the opposite stance and banned non-official harnesses on subscriptions — [manifest.build (2026-07-01)](https://manifest.build/blog/chatgpt-plus-tokens-third-party-harnesses/); [Simon Willison (2026-03-07)](https://simonwillison.net/2026/Mar/7/codex-for-open-source/)

### Inferences
- If kitchen runs the official `codex app-server` and lets users `account/login` with ChatGPT, OpenAI workers bill to the user's plan through the sanctioned client, which carries the least policy risk. Reimplementing Codex's backend request shape in kitchen (as OpenCode plugins do) works today but relies on informal tolerance.
- ChatGPT-plan auth only covers OpenAI models. Claude subscription auth cannot be reused through Codex.

### Gaps
- I found no formal written OpenAI policy document (only executive posts and secondary reporting). The policy could change.

## Overall fit assessment for kitchen (inference)
- **Use Codex as a worker adapter, not as kitchen's base.**
  - Strengths: the app-server JSON-RPC (threads, turns, items, approvals, steer, interrupt, fork and resume, generated TS types) maps almost 1:1 onto kitchen's worker-event and "needs me" model. Sandboxing, approvals and compaction are best in class. ChatGPT-plan auth is a real user benefit.
  - Weaknesses: the engine is Responses-API-only (non-OpenAI models need a gateway and run with GPT-tuned tools and prompts). Its own multi-agent support is single-provider. The protocol has no semver guarantee, and the codebase is huge and fast-moving.
- **Recommended architecture:** kitchen defines a provider-neutral `WorkerAdapter` (start, steer, interrupt, approve, events). Implement it with a `codex app-server` adapter for OpenAI and ChatGPT-plan jobs (pinned binary, `generate-ts` types, `dynamicTools` for manager callbacks), and with Claude Agent SDK, Gemini CLI or a paved-road library (e.g. Vercel AI SDK / pi-style loop) adapters for other providers. Do not fork codex-rs.
