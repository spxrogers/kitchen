# Other Open-Source Coding-Agent Harnesses (excluding opencode, Codex CLI, pi) — as of Oct 2026

Method note: star counts and repo metadata come from the GitHub search API on 2026-10-03 (via GitHub MCP `search_repositories`). Other facts come from repo READMEs, official docs and blogs fetched the same day. Claims marked **[training-knowledge, unverified]** come from my pre-2026 background knowledge and were not re-confirmed in this pass. The README fetches went through a summarizing model, so fine-grained details like license fine print should get a final check against the repo before a decision.

## Per-harness survey: license, runtime, backing, maturity, providers, embeddability, architecture, multi-agent, persistence, fit

### Takeaway
OpenHands' Software Agent SDK is the only candidate here built as an embeddable, event-sourced agent runtime with a REST/WebSocket server and pluggable workspaces, which is what kitchen needs underneath. But it is Python, so it does not match a Bun/TS daemon. Qwen Code (`qwen serve`, TS/Python/Java SDKs), Cline (Node SDK + `--acp` + headless JSON), Goose (Rust, ACP server, recipes, subagents) and Gemini CLI (ACP, subagents, A2A) are all good *worker* candidates over ACP or headless JSON. None of them provides a Kitchen-Manager-style orchestrator over heterogeneous workers in worktrees. The closest product in that space is Orca, a GUI that wraps CLI agents in terminals.

### Cited Findings

**Gemini CLI (google-gemini/gemini-cli)**
- Apache-2.0, TypeScript, npm `@google/gemini-cli`, 107.2k stars, 14.7k forks, created Apr 2025, pushed 2026-10-03 — [GitHub](https://github.com/google-gemini/gemini-cli)
- Headless mode: `--output-format json` and `--output-format stream-json` (newline-delimited JSON events). Also MCP servers, custom extensions/commands, checkpointing to save/resume conversations, sandboxing plus trusted-folder controls — [GitHub README](https://github.com/google-gemini/gemini-cli)
- Model support is mainly Gemini (Gemini 3, 2.5-flash) via `-m` — [GitHub README](https://github.com/google-gemini/gemini-cli)
- Listed as a **native ACP agent** on the ACP site — [ACP agents list](https://agentclientprotocol.com/overview/agents). It was the first reference ACP agent alongside Zed's 2025 launch **[training-knowledge, unverified]**
- Subagents launched 2026-04-15. Each runs with its own context window, system instructions and tool set. There are 3 built-in subagents, custom agents as Markdown + YAML frontmatter files, and parallel execution — [Google Developers Blog](https://developers.googleblog.com/en/subagents-have-arrived-in-gemini-cli/); [blog summary](https://codex.danielvaughan.com/2026/04/15/gemini-cli-subagents-launch/)
- Remote subagents over **A2A** (experimental). A2A HTTP auth and authenticated agent-card discovery shipped in v0.33.0 (2026-03-11) — [Gemini CLI remote agents docs](https://geminicli.com/docs/core/remote-agents/); [changelog](https://www.geminicli.com/docs/changelogs)
- Fit: **worker (via ACP/stream-json) + borrow-from.** It is mostly tied to Gemini models, so it is a poor base for a multi-provider core. It is a strong "Gemini worker" and a reference for subagent definitions (Markdown + frontmatter) and A2A remote delegation.

**Goose (aaif-goose/goose, formerly block/goose)**
- Apache-2.0, Rust, 54.9k stars. The project now lives under the **Agentic AI Foundation (AAIF) at the Linux Foundation** and the repo moved from `block/goose` to `aaif-goose/goose`. Repo topics include `acp` and `mcp` — [GitHub](https://github.com/aaif-goose/goose)
- 15+ providers (Anthropic, OpenAI, Google, Ollama, OpenRouter, Azure, Bedrock), "70+ extensions" via MCP, desktop app + CLI + API. It can also **use** Claude/ChatGPT/Gemini subscriptions "through ACP providers", so Goose acts as an ACP *client* to other agents as well as an ACP agent — [GitHub README](https://github.com/aaif-goose/goose)
- Works as an ACP server for Zed/JetBrains/VS Code. Recent releases added streamable-HTTP compliance, slash commands over ACP, session pagination, image replay, context-window forwarding and per-session system prompts. v1.36/1.37 shipped around June 2026 — [AAIF blog](https://aaif.io/blog/goose-doubles-down-on-open-in-latest-two-releases)
- Subagents run in parallel (code review, research, file processing). **Recipes** are declarative YAML workflows (extensions, model, instructions, parameters, subrecipes) that can run in CI — [goose docs: subagents](https://goose-docs.ai/docs/guides/context-engineering/subagents); [AAIF blog](https://aaif.io/blog/goose-doubles-down-on-open-in-latest-two-releases)
- A local server (`goosed`) backs the desktop app **[training-knowledge, unverified for 2026 API shape]**
- Fit: **worker + borrow-from.** Rust core with foundation governance makes it a strong ACP worker. Its use of ACP in both directions (client to other agents, server to editors) is the closest existing precedent for "ACP as kitchen's worker protocol". The recipe YAML is a useful model for kitchen "job tickets".

**Crush (charmbracelet/crush)**
- **FSL-1.1-MIT** (Functional Source License, converts to MIT later; not OSI-open at release), Go, 28.5k stars — [GitHub](https://github.com/charmbracelet/crush)
- Providers come from Charm's open "Catwalk" model database (OpenAI, Anthropic, Gemini, many others, plus custom configs). MCP over stdio/HTTP/SSE. LSP integration for code context — [GitHub README](https://github.com/charmbracelet/crush)
- **`crush serve`** server mode: several clients connect to a shared backend, grouped by working directory into "workspaces" that share session history and state over SSE streams. Permission prompts are on by default, and `--yolo` turns them off — [GitHub README](https://github.com/charmbracelet/crush)
- The README excerpt mentioned neither ACP nor subagents, and Crush does **not** appear on the ACP agents list — [ACP agents list](https://agentclientprotocol.com/overview/agents)
- Fit: **reference-only (TUI/UX + client/server split).** The license and Go stack rule out building on it. Its multi-client `serve` architecture and Bubble Tea TUI are good references for kitchen's daemon + TUI split.

**Aider (Aider-AI/aider)**
- Python, 49.4k stars, created May 2023, still active (pushed 2026-10-03) — [GitHub](https://github.com/Aider-AI/aider). Releases come roughly every two weeks. It has a single maintainer (Paul Gauthier), so bus-factor risk is high; 6.8M pip installs as of Apr 2026 — [codemyspec review 2026](https://codemyspec.com/blog/aider-review-2026) (secondary source)
- Apache-2.0. Multi-provider via **LiteLLM**. Edit formats (diff/whole/udiff), repo-map (tree-sitter), auto-commit on every edit, scriptable through `--message` and a Python API that is not officially supported **[training-knowledge, unverified]**
- Not on the ACP agents list. No native subagents or server mode found — [ACP agents list](https://agentclientprotocol.com/overview/agents)
- Fit: **reference-only / borrow ideas** (repo-map, edit formats, git auto-commit discipline). It is a pair-programming loop, not an embeddable agent runtime.

**Cline (cline/cline)**
- Apache-2.0 (© 2026 Cline Bot Inc.), TypeScript, 69.8k stars. Now described as "Autonomous coding agent as an **SDK**, IDE extension, or CLI assistant". Ships a Node.js SDK, CLI (TUI + headless), VS Code, JetBrains and a desktop app — [GitHub](https://github.com/cline/cline)
- Providers: Anthropic, OpenAI, Gemini, OpenRouter, Bedrock, Azure, Cerebras, Groq, Ollama, any OpenAI-compatible endpoint. Has MCP and a plugin system, plan/act modes, checkpoints, coordinator agents that delegate to specialists with persisted state, and cron-scheduled agents — [GitHub README](https://github.com/cline/cline)
- **Cline CLI 2.0 (2026-02-14)**: headless execution, NDJSON output, command-permission guardrails, `--acp` flag (ACP agent), and parallel isolated instances (one process per agent) — [Cline blog](https://cline.bot/blog/introducing-cline-cli-2-0); [DevOps.com](https://devops.com/cline-cli-2-0-turns-your-terminal-into-an-ai-agent-control-plane/). Listed as a native ACP agent — [ACP agents list](https://agentclientprotocol.com/overview/agents)
- "Cline Kanban" + worktrees for parallel work is mentioned only in a vendor-adjacent blog — [fast.io](https://fast.io/resources/cline-git-worktrees-parallel-agents/) (low confidence)
- Fit: **strong worker; possible borrow-from for the TS SDK.** Same language family as kitchen (TS/Node). The SDK could in principle serve as a TS agent loop. Kitchen would need to check that it runs on Bun and how tightly it is coupled to the Cline account/product.

**Roo Code (RooCodeInc/Roo-Code) — DROPPED**
- Shut down on 2026-05-15 (extension, Cloud, Router). The repo is archived and the company pivoted to "Roomote", a cloud agent. Users were pointed to the community fork ZooCode or to Cline — [Distill Intelligence](https://www.distillintelligence.com/news/roomote); [The New Stack](https://thenewstack.io/?p=22820574)
- One-line: irrelevant now. It is an archived VS Code extension, and its lineage lives on in Kilo/ZooCode.

**Kilo Code (Kilo-Org/kilocode)**
- TypeScript, 27.5k stars. Fetched README reports "MIT with attribution required for commercial use" (unusual; verify the LICENSE file directly) — [GitHub](https://github.com/Kilo-Org/kilocode)
- **Kilo CLI is a fork of OpenCode**, "enhanced to work within the Kilo agentic engineering platform". Ships VS Code, JetBrains, CLI and cloud (app.kilo.ai) versions. Agents: Code/Plan/Ask/Debug plus custom agents via a marketplace. `kilo run --auto` gives fully autonomous CI runs. Claims 500+ models through the Kilo gateway — [GitHub README](https://github.com/Kilo-Org/kilocode)
- Fit: **reference-only.** For building on, use upstream opencode (covered separately) rather than a product fork tied to the Kilo gateway.

**OpenHands (OpenHands/OpenHands) + Software Agent SDK (OpenHands/software-agent-sdk)**
- OpenHands app: 89.9k stars; repo moved from All-Hands-AI to the `OpenHands` org — [GitHub](https://github.com/OpenHands/OpenHands). Native ACP agent — [ACP agents list](https://agentclientprotocol.com/overview/agents)
- Software Agent SDK: **MIT**, Python (plus a TypeScript client), about 1.2k stars, 602 forks. Packages: `openhands-sdk`, `openhands-tools`, `openhands-workspace`, `openhands-agent-server` — [GitHub](https://github.com/OpenHands/software-agent-sdk)
- Architecture: a typed **event framework** (actions, observations, user messages, state updates) drives conversation state. The LLM interface is provider-agnostic with retry and telemetry. A **Condenser** compresses history. A **Security** layer assesses action risk before execution. Workspaces come as `LocalWorkspace`, `DockerWorkspace` and `RemoteAPIWorkspace`, and "same agent code works in both modes — just swap the workspace type." The Agent Server exposes **REST + WebSocket** endpoints for conversations, bash, files, events, desktop and VSCode, plus OpenAI-compatible Chat Completions/Responses endpoints — [OpenHands SDK arch overview](https://docs.openhands.dev/sdk/arch/overview); [SDK docs](https://docs.openhands.dev/sdk)
- LLM layer built on **LiteLLM** **[training-knowledge, unverified — the 2026 docs only say "provider-agnostic"]**. The OpenHands V1 SDK paper/announcement (late 2025) described event-sourced state and a delegation tool for sub-agents **[training-knowledge, unverified; the arch overview fetched did not mention sub-agent delegation]**
- Fit: **best architectural reference; build-on only if kitchen accepts a Python worker runtime.** Its event log + REST/WS agent-server + swappable workspace design is close to kitchen's plan, so kitchen could run `openhands-agent-server` per worker. Otherwise kitchen should borrow the event taxonomy, condenser and security-analyzer concepts.

**Qwen Code (QwenLM/qwen-code)**
- Apache-2.0, TypeScript, 28.3k stars. Forked from Gemini CLI v0.8.2 but has developed independently since Qwen Code v0.1. README references v0.22.0 — [GitHub](https://github.com/QwenLM/qwen-code)
- Multi-protocol providers: OpenAI, Anthropic, Gemini and Qwen APIs, plus any third-party or local model (Ollama/vLLM). **SDKs in TypeScript, Python and Java.** Headless via `qwen -p`. **ACP**, plus an experimental **`qwen serve` daemon** that lets several clients share agent connections over **HTTP + SSE**. SubAgents and "Agent Teams" are core features — [GitHub README](https://github.com/QwenLM/qwen-code). Native ACP agent — [ACP agents list](https://agentclientprotocol.com/overview/agents)
- Fit: **strong worker + borrow-from.** It is the most multi-provider of the Gemini-CLI lineage, it is TS, and its `serve` daemon + SDKs mirror the shape kitchen wants. The TS SDK is worth evaluating as an in-process worker driver.

**Other notable 2025–2026 entrants**
- **Orca (stablyai/orca)**: MIT, TypeScript, **84.3k stars**, created Mar 2026, YC-backed. "The ADE for working with a fleet of parallel agents." It fans one prompt across N agents, each in its **own git worktree**, then compares and merges the winner. It wraps "any CLI agent" (Claude Code, Codex, Cursor, Copilot, OpenCode, Pi, 40+) in terminals rather than talking to APIs. It also offers SSH remote worktrees and "unread state" for knowing when an agent "finishes or needs attention", plus mobile notifications — [GitHub](https://github.com/stablyai/orca). **Fit: the closest competitor/reference for kitchen's UX ("what needs me", worktrees). It is not an orchestrator *agent*: the human is the manager.**
- **herdr (herdrdev/herdr)**: Rust, 42k stars, "the runtime your coding agents live on". It is a terminal multiplexer/workspace manager for agents, with topics including agent-orchestration and tmux — [GitHub search](https://github.com/herdrdev/herdr). Reference-only (terminal-level orchestration).
- **Hermes Agent (NousResearch/hermes-agent)**: Python, 250.9k stars, a general self-improving personal agent, native ACP — [GitHub](https://github.com/NousResearch/hermes-agent); [ACP list](https://agentclientprotocol.com/overview/agents). It is a general assistant rather than a coding harness, so it is relevant only as a possible ACP worker.
- **Mistral Vibe (mistralai/mistral-vibe)**: Python, 5k stars, "Minimal CLI coding agent by Mistral", native ACP — [GitHub](https://github.com/mistralai/mistral-vibe); [ACP list](https://agentclientprotocol.com/overview/agents). Worker candidate only.
- **Kimi CLI**: the Python `MoonshotAI/kimi-cli` (11.4k stars) is **archived** and replaced by "Kimi Code CLI" (`MoonshotAI/kimi-code`). Kimi CLI is listed as a native ACP agent — [GitHub](https://github.com/MoonshotAI/kimi-cli); [ACP list](https://agentclientprotocol.com/overview/agents)
- **Codewhale (Hmbown/Codewhale)**: Rust, 41k stars, terminal coding agent with multi-agent/multi-model topics — [GitHub search](https://github.com/Hmbown/Codewhale). Not investigated further.
- **Open Interpreter**: now "a coding agent for open models like Kimi K3 and GLM 5.3", rewritten in Rust, with topic `acp` — [GitHub](https://github.com/openinterpreter/openinterpreter). Worker candidate only.
- **Docker cagent, Factory Droid, Junie, Kiro CLI, Augment, Copilot CLI, Cursor, Claude Agent (Claude Code adapter)** are all listed as ACP agents but are closed-source or vendor CLIs — [ACP list](https://agentclientprotocol.com/overview/agents). They matter to kitchen only as ACP workers.
- Dropped as out of scope (very popular repos, but not harnesses): obra/superpowers, ECC, claude-mem, codegraph and other "skills"/context add-ons; bytedance/deer-flow and ruvnet/ruflo (general multi-agent frameworks/swarms, not coding harnesses to embed) — [GitHub search](https://github.com/search?q=coding+agent)

**Summary table**

| Harness | License | Lang | Stars (Oct 2026) | Providers | Headless / server / SDK | ACP | Subagents / parallel | Fit for kitchen |
|---|---|---|---|---|---|---|---|---|
| Gemini CLI | Apache-2.0 | TS | 107k | Mostly Gemini | stream-json; npm pkg | Native | Subagents (Apr 2026), A2A remote | Worker; borrow subagent defs |
| Goose | Apache-2.0 | Rust | 55k | 15+, plus ACP providers | CLI/desktop/API (goosed) | Native (server + client) | Subagents, recipes | Worker; borrow recipes / ACP-client pattern |
| Crush | FSL-1.1-MIT | Go | 28k | Catwalk DB, many | `crush serve` multi-client | Not listed | Not mentioned | Reference (TUI, serve) |
| Aider | Apache-2.0* | Python | 49k | LiteLLM* | `--message` scripting | No | No | Reference (repo-map, git) |
| Cline | Apache-2.0 | TS | 70k | Many | Node SDK, headless NDJSON | Native (`--acp`) | Coordinator agents; parallel instances | Worker; evaluate SDK |
| Roo Code | — | TS | archived | — | — | — | — | Dropped (shut down May 2026) |
| Kilo Code | MIT+attrib? | TS | 27.5k | 500+ via Kilo gateway | CLI = opencode fork | (via opencode) | Modes | Reference; use upstream opencode |
| OpenHands SDK | MIT | Python | 90k app / 1.2k SDK | Provider-agnostic (LiteLLM*) | Python SDK, REST+WS agent-server | Native (app) | Delegation* | Best architecture reference; Python worker runtime |
| Qwen Code | Apache-2.0 | TS | 28k | OpenAI/Anthropic/Gemini/local | `-p`, `qwen serve` HTTP+SSE, TS/Py/Java SDKs | Native | SubAgents, Agent Teams | Worker; borrow serve/SDK |
| Orca | MIT | TS | 84k | Any CLI agent | Desktop/mobile/remote | Wraps CLIs | Parallel worktrees, "needs attention" | Competitor/UX reference |

(* = training-knowledge, unverified in this pass)

### Inferences
- No surveyed open-source harness ships an *LLM orchestrator agent* that manages heterogeneous external worker harnesses in worktrees and surfaces "what needs me". The parallel tools (Orca, herdr, Cline tmux) put the human in the manager role, and the in-harness subagents (Gemini, Goose, Qwen, Cline) delegate only to themselves. This is kitchen's differentiator, which argues for **owning the orchestrator/daemon** and plugging in existing harnesses as workers.
- OpenHands' design (typed events → state, workspace abstraction, REST/WS server) independently validates kitchen's append-only event log + seq replay plan.
- Kitchen's TS/Bun stack makes Cline's SDK, Qwen Code's TS SDK and the Gemini CLI core the realistic in-process candidates. All three would need Bun-compatibility testing (not verified).

### Gaps
- Exact latest release versions/dates for Gemini CLI, Cline, Crush, Kilo and OpenHands SDK. The fetched pages did not show them.
- Kilo's exact license wording ("MIT with attribution for commercial use") needs a direct LICENSE check.
- Whether the OpenHands SDK still uses LiteLLM and has a delegation tool in 2026. The arch overview did not say.
- Whether the Cline SDK and Qwen Code SDK run under Bun. Not tested.
- Crush ACP status: neither its README nor the ACP list shows support, but a recent addition can't be ruled out.

## Cross-cutting standards: ACP, AGENTS.md, MCP — and ACP as kitchen's worker protocol

### Takeaway
By Oct 2026 ACP has become the de facto "drive a coding agent" protocol: about 38 native agents are listed, including Gemini CLI, Goose, Cline, OpenHands, Qwen Code, OpenCode, Copilot, Cursor, Junie, Kiro and Mistral Vibe, with adapters for Codex CLI and pi. That makes it a credible **worker protocol** for kitchen. It covers sessions, prompting, streaming updates, permission requests, cancel and load/resume. Kitchen still needs its own layer above ACP for worktree provisioning, cost/usage, task metadata and cross-worker event persistence.

### Cited Findings
- ACP site lists native agents: AgentPool, Augment Code, AutoDev, Blackbox AI, Bub, Claude Agent, Claw Orchestrator, Cline, Code Assistant, Construct, crow-cli, Cursor, Docker cagent, fast-agent, Factory Droid, fount, Gemini CLI, GitHub Copilot, Goose, Hermes Agent, Junie, Kaagum, Kimi CLI, Kiro CLI, localharness, Minion Code, Mistral Vibe, OpenClaw, OpenCode, OpenHands, Poolside, Qoder CLI, Qwen Code, Raxol, siGit Code, Stakpak, stdio Bus, VT Code. Codex CLI and Pi work through adapters (pi-acp) — [ACP agents](https://agentclientprotocol.com/overview/agents)
- The protocol repo now lives in its own org: `agentclientprotocol/agent-client-protocol`, Rust, 4.4k stars, "A protocol for connecting any editor to any agent" — [GitHub](https://github.com/agentclientprotocol/agent-client-protocol)
- ACP is JSON-RPC 2.0. Agent methods: `initialize` (capability and version negotiation), `authenticate`, `session/new`, `session/load`, `session/prompt`, `session/cancel`. Client methods: `session/update` notifications, `session/request_permission`, and optional fs/terminal capabilities. Extensions use `_`-prefixed methods and `_meta` fields — [ACP protocol overview](https://agentclientprotocol.com/protocol/overview)
- Transport is stdio subprocess by default **[training-knowledge]**. Goose reports "streamable HTTP compliance" for ACP, which suggests remote/HTTP transports are emerging — [AAIF blog](https://aaif.io/blog/goose-doubles-down-on-open-in-latest-two-releases). Qwen Code's `qwen serve` gives multi-client HTTP+SSE — [Qwen Code README](https://github.com/QwenLM/qwen-code)
- Goose acts as an ACP **client** too, using Claude/ChatGPT/Gemini subscriptions "through ACP providers", which is effectively "other agents as workers" — [Goose README](https://github.com/aaif-goose/goose)
- Orca does **not** use ACP. It wraps 40+ CLI agents through terminals — [Orca README](https://github.com/stablyai/orca)
- MCP is supported by every surveyed harness (Gemini CLI, Goose, Crush, Cline, OpenHands, Qwen Code) — see the per-harness sources above
- AGENTS.md: `openai/agents.md` lives at the AAIF alongside MCP and goose **[training-knowledge, unverified: AAIF founded Dec 2025 with MCP, goose, AGENTS.md]**. I did not re-verify per-harness AGENTS.md support in this pass.

### Inferences
- **ACP as kitchen's worker protocol is viable and recommended.** Kitchen's daemon acts as an ACP *client*, spawning `gemini --acp`-style (exact flags vary per harness), `goose acp`, `cline --acp`, `qwen --acp`, opencode, and Codex/pi via adapters, each with `cwd` = a worktree. `session/update` streams map onto kitchen's event log. `session/request_permission` maps directly onto "what needs me". `session/load` supports resume.
- Gaps kitchen must fill above ACP: worktree lifecycle, model/provider selection (not standardized in ACP; harness-specific config or `_meta`), token/cost reporting, structured "done/blocked" task outcomes, and persistence across daemon restarts (ACP sessions are per-process). Kitchen should namespace these as `_kitchen/*` extension methods or `_meta`.
- A thin first-party worker (e.g., an AI-SDK-based TS loop that also speaks ACP) gives kitchen one fully controlled worker type, while third-party harnesses plug in through the same interface.

### Gaps
- The exact ACP protocol version as of Oct 2026, and whether session listing or multiple concurrent sessions per connection are standardized. The fetched overview did not say.
- Whether a remote/HTTP ACP transport is officially in the spec or only implemented by Goose.
- An authoritative list of harnesses that read AGENTS.md in 2026.
