# pi (Mario Zechner / Earendil) — minimal open-source coding harness and layered TS libraries

Research date: 2026-10-03. Primary method: shallow clone of the repo at HEAD `83692682` (commit 2026-10-03), reading package READMEs/docs/CHANGELOGs directly, plus npm registry and web sources. Repo files are cited via their GitHub URLs on earendil-works/pi (badlogic/pi-mono redirects there).

**Biggest change since the 2025 coverage most people know:** in April 2026, Earendil Inc. (Armin Ronacher's company) acquired pi. The repo moved from `badlogic/pi-mono` to `earendil-works/pi`. The npm scope moved from `@mariozechner/*` to `@earendil-works/*`, and the old scope stopped at 0.73.1 in May 2026. pi hit **1.0.0 on 2026-10-01** and 1.0.1 on 2026-10-03. The monorepo has also grown well past the original four packages: it now includes a durable SQLite/JSONL-backed harness runtime, a CBOR remote protocol, a server, and a composition runtime called "chord". Anything you read about "@mariozechner/pi-ai" is about the old scope and API.

## Identity, license, runtime, packages, popularity, cadence, adopters

### Takeaway
"pi" here means the MIT-licensed TypeScript agent toolkit and coding CLI at pi.dev, created by Mario Zechner (badlogic, author of libGDX) and owned by Earendil Inc. since April 2026. It is very popular (tens of thousands of GitHub stars, possibly more than 100k) and ships releases at a very fast pace, often several a week. OpenClaw is the best-known product that embeds it, through the in-process SDK.

### Cited Findings
- README tagline: "Pi is a minimal, extensible agent harness that you can make your own… Pi ships with powerful defaults but skips features like sub-agents and plan mode." It names OpenClaw as "a real-world integration." — [README](https://github.com/earendil-works/pi/blob/main/README.md)
- License: MIT. The LICENSE file reads "Copyright (c) 2025 Mario Zechner", and every package.json says MIT. The one exception is `pi-evals`, whose package.json has no license field. — [repo LICENSE](https://github.com/earendil-works/pi)
- Runtime: Node.js ≥ 22.19 is required. The release process smoke-tests "isolated npm and Bun installs", and pi-ai ships a `bun-oauth.ts` module. pi-durable's portable SQLite/JSONL storage cores "run without Node APIs, for example on Bun or in Cloudflare Durable Objects". — [README](https://github.com/earendil-works/pi/blob/main/README.md), [pi-durable README](https://github.com/earendil-works/pi/blob/main/packages/durable/README.md)
- Packages at HEAD are all versioned 1.0.1. Responsibilities, taken from each package.json description:
  - `@earendil-works/pi-ai`: "Unified LLM API with automatic model discovery and provider configuration"
  - `@earendil-works/pi-agent-core`: "General-purpose agent with transport abstraction, state management, and attachment support"
  - `@earendil-works/pi-coding-agent`: "Coding agent CLI with read, bash, edit, write tools and session management"
  - `@earendil-works/pi-tui`: "Terminal User Interface library with differential rendering"
  - `@earendil-works/pi-durable` (experimental): "Durable conversation, task, and document runtime"
  - `@earendil-works/chord`: "Application composition runtime for services, replicated state, RPC, and plugins". It does not depend on any other pi package.
  - `@earendil-works/pi-protocol`: "Transport-neutral CBOR protocol for remote pi sessions"
  - `@earendil-works/pi-client`: client for remote pi sessions over framed CBOR
  - `@earendil-works/pi-server`: "experimental server package"
  - `@earendil-works/pi-mcp`: standalone MCP client
  - `@earendil-works/pi-codemode`: "Sandboxed JavaScript execution where the only capability is calling injected tools"
  - `@earendil-works/pi-telemetry`
  - `pi-evals`
  
  Slack/chat automation lives in the separate repo `earendil-works/pi-chat`. — [README packages table](https://github.com/earendil-works/pi/blob/main/README.md)
- pi-ai's direct dependencies are pinned exactly: `@anthropic-ai/sdk` 0.129.0, `openai` 7.19.0, `@google/genai` 2.21.0, `@aws-sdk/client-bedrock-runtime`, `typebox` 1.3.27, `partial-json`, and proxy agents. — npm registry (`npm view @earendil-works/pi-ai dependencies`)
- Release cadence for `@earendil-works/pi-coding-agent`: 56 npm versions since 0.74.0 (about May 2026). Recent ones: 0.86.1 (Sep 20), 0.87.0 (Sep 21), 0.87.1 (Sep 22), 0.99.0/0.99.1 (Sep 29), 0.99.2 (Sep 30), 1.0.0 (Oct 1), 1.0.1 (Oct 3). The CHANGELOG reaches back to 0.10.0 (2025-11-25). The old `@mariozechner/pi-ai` scope was created 2025-08 and last published 0.73.1 on 2026-05-07. — npm registry; [coding-agent CHANGELOG](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/CHANGELOG.md)
- Commit authors since 2026-07-01 (shallow clone, 244 commits): Armin Ronacher 115, Mario Zechner 65, David Brailovsky 33, Christian Klotz 15, then a long tail. Upstream reports about 6,709 commits on main. — git log of clone; [GitHub repo page](https://github.com/earendil-works/pi)
- Stars: the GitHub page fetched 2026-10-03 showed about 112k stars and 14.2k forks. That number came through an LLM summarizer and could not be verified with the API, which was blocked in this session. A wiki source says "over sixty-two thousand stars" as of June 2026. — [GitHub](https://github.com/earendil-works/pi); [ai.miraheze wiki](https://ai.miraheze.org/wiki/Pi.dev)
- Contribution policy: "All issues and PRs from new contributors are auto-closed by default." Maintainers reopen worthwhile ones. Contributors get rights through `lgtmi`/`lgtm` replies. Issues filed Friday to Sunday are not guaranteed review. "PRs that bloat the core will likely be rejected." — [CONTRIBUTING.md](https://github.com/earendil-works/pi/blob/main/CONTRIBUTING.md)
- Supply-chain posture: direct dependencies are pinned exactly, `.npmrc` uses `min-release-age=2`, installs use `--ignore-scripts`, there is a lifecycle-script allowlist, and the pi.dev installer pins transitive dependencies. Since 1.0.1 the npm package no longer ships an npm-shrinkwrap. — [README](https://github.com/earendil-works/pi/blob/main/README.md), [CHANGELOG 1.0.1](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/CHANGELOG.md)
- Earendil acquisition. On 2026-04-08 Earendil said it had acquired pi and that Mario had joined. The company describes itself as backed by Accel and Balderton and announced "Lefos" at the same time. — [Earendil announcement reflection](https://earendil.com/posts/announcement-reflection/); [Agent Wars](https://agent-wars.com/news/2026-04-08-pi-agent-creator-joins-earendil)
- Secondary sources add detail: Earendil is a public benefit corporation co-founded by Armin Ronacher and Colin Daymond Hanna. Zechner became a "major Earendil stakeholder", and the repo moved from badlogic/pi-mono to earendil-works/pi. Mario's own post was titled "I've sold out". — [ai.miraheze wiki](https://ai.miraheze.org/wiki/Pi.dev) (secondary)
- Licensing RFC 0015 (2026-03-30): "Pi remains MIT licensed and that will not change." Earendil also plans Fair Source commercial products with Delayed Open Source Publication, some permanently proprietary or cloud-only features, and trademark enforcement as its main control mechanism. — [RFC 0015](https://rfc.earendil.com/0015/)
- OpenClaw imports pi's `createAgentSession()` in-process rather than using RPC. Its "Pi Embedded Runner" (`src/agents/pi-embedded-runner/`) adds custom tools, a system prompt per channel, multi-account auth-profile rotation with failover, and model switching. — [OpenClaw docs: pi](https://docs.openclaw.ai/pi)
- Mario has also given a talk on "embedding pi coding agent" at AI Engineer / AIDevCon London 2026. — [tessl talk outline](https://tessl.io/registry/ainativedev/latest-aidevcon-speakers-london-2026/0.1.6/files/talk-luebken-embedding-pi-coding-agent/outline.md); [ai.engineer speaker page](https://ai.engineer/speakers/mario-zechner)

### Inferences
- **Bus factor:** before April 2026, pi was close to a one-person project. Today it is maintained by a funded company, and the most frequent recent committer is Armin Ronacher, not Mario. That lowers the single-maintainer risk. It adds a different risk: Earendil's commercial strategy (Fair Source/proprietary layers, trademark control) decides where the project goes.
- The project churns fast: many breaking renames, about 3 minor versions a week, and a scope migration. 1.0 is brand new. Pin exact versions and expect migration work.
- **Name ambiguity:** "Pi" is also Inflection AI's consumer chatbot, which is not a coding harness. I found no other notable coding harness called "pi". In coding-agent discussions, "pi" almost always means this project (pi.dev).

### Gaps
- Exact current star and contributor counts could not be verified because the GitHub API was blocked in this session. The 112k figure comes from a summarized page fetch.
- I could not read Mario's "I've sold out" post or Armin's "Mario and Earendil" post directly.
- I found no list of adopters beyond OpenClaw. pi-chat and Earendil's cloud platform/Lefos are mentioned but were not researched.

## pi-ai: providers, tools, streaming, thinking, handoff, cost, OAuth/ToS

### Takeaway
pi-ai is a mature unified LLM layer. It supports about 35 providers plus any OpenAI-compatible endpoint, with typed streaming events, partial-JSON tool-call streaming, a unified reasoning knob, cross-provider handoff, token and cost tracking, and pluggable OAuth/credential stores. It is the most reusable piece for kitchen.

### Cited Findings
- Tagline: "Unified LLM API with provider collections, automatic auth resolution, token and cost tracking, and simple context persistence and hand-off to other models mid-session." The chat catalog includes only models that support tool calling. — [pi-ai README](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md)
- Providers listed: OpenAI, Ant Ling, Azure OpenAI Responses, OpenAI Codex (legacy, ChatGPT subscription), Radius, TypeSafe, DeepSeek, NVIDIA NIM, Anthropic, Google, Vertex AI, Mistral, Groq, Cerebras, Cloudflare AI Gateway, Cloudflare Workers AI, xAI, OpenRouter, Vercel AI Gateway, ZAI, MiniMax, Together, Baseten, Hugging Face, Moonshot, GitHub Copilot, Amazon Bedrock, OpenCode Zen/Go, Fireworks, Kimi For Coding, Meta, Qwen Token Plan, Xiaomi MiMo, and "Any OpenAI-compatible API: Ollama, vLLM, LM Studio". — [pi-ai README](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md)
- API shape: `createModels()` returns a Models collection. You register providers with `setProvider(anthropicProvider())`, then call `getModel(provider,id)`, `stream`/`complete` (provider-specific options), and `streamSimple`/`completeSimple` (unified `reasoning: 'minimal'…'high'`). Provider factories are tree-shakeable, and provider SDKs load lazily on first request. `createProvider()` builds custom providers, and a "Faux Provider" exists for tests. The model catalog overlays newer data from pi.dev at runtime. — [pi-ai README](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md); [README Nix section](https://github.com/earendil-works/pi/blob/main/README.md)
- Streaming event protocol:
  - Order is `start → (text|thinking|toolcall)_(start|delta|end)* → done | error`.
  - `toolcall_delta` carries partial parsed arguments.
  - `toolcall_end` delivers a complete but not schema-validated call. `validateToolArguments()` is available, as is constrained sampling for tools.
  - Events for different blocks can interleave, so consumers must key on `contentIndex`.
  - `done.reason` is one of stop, length, or toolUse. `error.reason` is error or aborted and includes the partial message.
  - There are AbortSignal-based aborts and "Continuing After Abort".
  
  — [pi-ai README §Complete Event Reference](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md)
- Tools are defined with TypeBox schemas, and "Strict mode" defaults vary by provider (fix in Sep 2026: "default unknown providers to non-strict tools"). — [pi-ai README §Tools](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md); git log
- Cross-provider handoff: user and tool-result messages pass through unchanged. Same-provider assistant messages are kept as-is. Assistant messages from a different provider "have their thinking blocks converted to text with `<thinking>` tags". Tool calls, images, and aborted partial messages also survive the switch. Context objects are plain serializable data ("Context Serialization"). — [pi-ai README §Cross-Provider Handoffs](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md)
- System prompt and tools live in the transcript as `SystemMessage` entries: a leading system message plus later patch messages with `content` or named `sections`. In 1.0.1, Anthropic tools defined mid-conversation are defined inline so the prompt cache survives. — [agent-core README §Agent State](https://github.com/earendil-works/pi/blob/main/packages/agent/README.md); [CHANGELOG 1.0.1](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/CHANGELOG.md)
- Cost: pricing tiers come from models.dev data (e.g. a Bedrock >272k-token tier fix in 1.0.1). Mario's 2025 post called cost tracking "best-effort". — [CHANGELOG 1.0.1](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/CHANGELOG.md); [Mario's blog post](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- OAuth providers: Anthropic (Claude Pro/Max), OpenAI "Sign in with ChatGPT", OpenAI Codex (legacy), GitHub Copilot, and OpenRouter (PKCE that mints an API key). Each has `login(interaction)`, `refresh`, and `toAuth`. Refresh is automatic "under a credential-store lock, so concurrent requests and processes cannot double-refresh". The `CredentialStore` is pluggable. Also supported: ambient credentials for Vertex and Bedrock, and Anthropic workload identity federation. — [pi-ai README §OAuth Providers](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md); [providers.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md)
- **ToS:**
  - Anthropic blocked subscription OAuth tokens outside official apps on Jan 9, 2026 and reversed after pushback.
  - On Feb 19, 2026 Anthropic updated its docs to say third-party use of OAuth tokens violates its terms.
  - On April 4, 2026 Anthropic enforced the block for third-party harnesses such as OpenClaw. Users now need API keys or pay-as-you-go "Extra Usage".
  
  — [VentureBeat](https://venturebeat.com/ai/anthropic-cracks-down-on-unauthorized-claude-usage-by-third-party-harnesses/); [Gigazine](https://gigazine.net/gsc_news/en/20260220-anthropic-third-party-block); [falcao.org](https://falcao.org/posts/anthropic-claude-access-crackdown-ecosystem-fallout/) (secondary)
- pi's docs do not mention any ToS caveat. They still list Anthropic Claude Pro/Max OAuth, and 1.0.0 added an "Anthropic copy code login". — [CHANGELOG 1.0.0](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/CHANGELOG.md)

### Inferences
- For kitchen, pi-ai covers most of what a "multi-provider per job" design needs: model catalog, auth, streaming normalization, and handoff.
- Kitchen should not depend on Claude subscription OAuth for workers. It violates Anthropic's terms per the Feb 2026 docs, and enforcement since April 2026 can block it.
- OpenAI's ChatGPT sign-in and Copilot OAuth appear to be officially supported paths. Their ToS for third-party use is not verified here.

### Gaps
- I did not verify the current Anthropic policy text (Oct 2026) or what pi's Anthropic OAuth login does after enforcement, for example whether it routes to Extra Usage.

## Agent core: loop, events, state, queues, abort

### Takeaway
`pi-agent-core` provides a small, stateful `Agent` class built on pi-ai. It has a typed event stream, steering and follow-up queues, per-tool parallel or sequential execution, before/after tool hooks, and abort. Persistence is left to the host, which suits a kitchen daemon that owns its own event log.

### Cited Findings
- Message flow: `AgentMessage[] → transformContext() → convertToLlm() → Message[] → LLM`. AgentMessage can include custom app message types via declaration merging, and these are filtered or converted before LLM calls. — [agent-core README](https://github.com/earendil-works/pi/blob/main/packages/agent/README.md)
- Events:
  - `agent_start`, `agent_end`, `turn_start`, `turn_end` (one LLM call plus its tool executions)
  - `message_start`, `message_update` (assistant only, carries the pi-ai delta), `message_end`
  - `tool_execution_start`, `tool_execution_update`, `tool_execution_end`
  
  `subscribe()` listeners are awaited in registration order. — [agent-core README §Event Types](https://github.com/earendil-works/pi/blob/main/packages/agent/README.md)
- State: `{model, thinkingLevel, tools, messages, isStreaming, streamingMessage, pendingToolCalls, errorMessage}`. `prepareRequest` runs before every provider request, for example to load canonical persisted context. `finishTurn` can end a run (it replaced `shouldStopAfterTurn`). — [agent-core README](https://github.com/earendil-works/pi/blob/main/packages/agent/README.md); [CHANGELOG](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/CHANGELOG.md)
- Steering and follow-up: `agent.steer(msg)` is injected after the current tool calls finish. `agent.followUp(msg)` is injected only when there are no more tool calls and no steering messages. Both queues have `"one-at-a-time"` or `"all"` modes and clear methods. Control methods are `agent.abort()` and `await agent.waitForIdle()`. — [agent-core README §Steering and Follow-up](https://github.com/earendil-works/pi/blob/main/packages/agent/README.md)
- Tools: `AgentTool` has a TypeBox schema, `execute(toolCallId, params, signal, onUpdate)`, and `executionMode` sequential or parallel per tool. `beforeToolCall`/`afterToolCall` hooks are available. `streamProxy` lets the LLM calls run through a backend. A low-level loop API exists without the Agent class. The example `examples/mcp-codemode` wraps MCP tools and a codemode tool. — [agent-core README](https://github.com/earendil-works/pi/blob/main/packages/agent/README.md)

### Inferences
- The event vocabulary lines up closely with what kitchen would record in an append-only log. A daemon could subscribe to each worker's Agent and write events with a `seq`.
- Steering and follow-up queues give the Kitchen Manager a ready mechanism to redirect workers.

### Gaps
- None significant for the in-process API. Performance and limits under many concurrent Agents in one process were not evaluated.

## Coding agent: tools, prompt, permissions, sessions, compaction, extensions, modes

### Takeaway
pi-coding-agent keeps a deliberately small core: 4 default tools, a short system prompt, YOLO permissions with containerization recommended, and JSONL tree sessions with branching and compaction. Almost everything else is done through TypeScript extensions. It offers interactive, print/JSON, RPC (JSONL over stdio), and in-process SDK modes, and since 2026 it ships built-in MCP and "codemode".

### Cited Findings
- Default tools are `["read","bash","edit","write"]`. `grep`, `find`, `ls`, and `powershell` also exist and can be enabled via `defaultTools`/`--tools`. — `packages/coding-agent/src/core/agent-session.ts` line ~3604; [tools dir](https://github.com/earendil-works/pi/tree/main/packages/coding-agent/src/core/tools)
- System prompt: the Nov 2025 post says it is under 1,000 tokens. The current `system-prompt.ts` source file is about 9.5 KB, including code, so the prompt text is still small, but I did not measure it exactly. Skills are loaded on demand: descriptions go in the prompt and full instructions load when needed. — [Mario's blog post](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/); [how-pi-works.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/how-pi-works.md)
- Permissions: "Pi does not include a built-in permission system for restricting filesystem, process, network, or credential access." For isolation the README suggests three options: the Gondolin extension (tools run in a local Linux micro-VM while auth stays on the host), plain Docker, or OpenShell. "Project trust" is resolved before project settings and resources load. Example extensions include `permission-gate.ts`, `protected-paths.ts`, `confirm-destructive.ts`, and `sandbox`. — [README](https://github.com/earendil-works/pi/blob/main/README.md); [how-pi-works.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/how-pi-works.md); [examples/extensions](https://github.com/earendil-works/pi/tree/main/packages/coding-agent/examples/extensions)
- Sessions:
  - JSONL files at `~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl`.
  - Entries form a tree through `id`/`parentId`, which allows "in-place branching without creating new files".
  - Format versions: v1 linear, v2 tree, v3 current, with automatic migration.
  - Fork and clone copy history into a new file. `/tree` navigates branches.
  
  — [session-format.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/session-format.md); [how-pi-works.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/how-pi-works.md)
- Compaction: auto-compaction runs at a threshold or on `/compact`, and inserts a summary entry while the original entries stay in the tree. Branch summarization preserves context when switching branches. File operations are tracked cumulatively. Extension hooks include `session_before_compact` and custom compaction. — [compaction.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/compaction.md)
- Extensions are TypeScript modules loaded in-process. They register tools, commands, shortcuts, providers, event handlers, renderers, and TUI. Hook events seen in `extensions/types.ts`:
  - session: `session_start`, `session_before_compact`, `session_before_fork`, `session_before_tree`, `session_shutdown`
  - agent and turn: `before_agent_start`, `agent_start`, `agent_end`, `agent_settled`, `turn_start`, `turn_end`
  - context and input: `context`, `context_edit`, `input`
  - tools and provider: `tool_call`, `before_provider_request`, `before_provider_headers`, `after_provider_response`
  
  Skills, prompt templates, and themes are bundled as "Pi packages" via npm or git. — [how-pi-works.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/how-pi-works.md); [extensions.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)
- Modes: interactive TUI (fullscreen by default since 1.0.0), print mode, JSON mode (agent events as JSONL), and RPC mode (`pi --mode rpc --no-session`). RPC takes JSONL commands on stdin and writes responses, session events, and extension-UI records to stdout, with `id` correlation. A typed `RpcClient` is exported. The SDK is `createAgentSession()` → `session.prompt()`, with SessionManager, AuthStorage, and ModelRegistry. "All interfaces use the same agent and session mechanisms." — [rpc.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/rpc.md); [sdk.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md); [how-pi-works.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/how-pi-works.md)
- **Stance reversal on MCP:** the Nov 2025 post rejected MCP ("Playwright MCP costs 13.7k tokens"). As of 1.0.x, pi has built-in MCP (`.pi/mcp.json`, `/mcp`, OAuth, per-tool "exposure"), plus "codemode", where the model writes JS that calls tools; 1.0.0 cut its prompt tokens by about 40%. — [CHANGELOG 1.0.0/1.0.1](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/CHANGELOG.md); [Mario's blog post](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- Mario publishes his own pi-mono work sessions as an HF dataset and encourages others to share theirs (`pi-share-hf`). — [README](https://github.com/earendil-works/pi/blob/main/README.md)

### Inferences
- RPC mode is a ready-made worker protocol. A kitchen daemon could run `pi --mode rpc` in each git worktree and get process isolation, all of pi's providers and tools, and a JSONL event stream to persist.
- The cost of that approach: two session stores (pi's JSONL plus kitchen's SQLite) and a dependency on pi's fast-changing RPC schema.

### Gaps
- Exact system prompt token count today is unverified.
- The RPC schema stability guarantee after 1.0 is not stated anywhere I found.

## Multi-agent / subagent stance

### Takeaway
The CLI core still deliberately omits subagents and plan mode, citing their lack of visibility. Subagents are offered as an example extension that spawns separate `pi` processes. The newer experimental pi-durable runtime, however, has first-class owned child conversations, background tasks, and a task graph. That is very close to kitchen's Manager→workers model.

### Cited Findings
- Mario on sub-agents: "You have zero visibility into what that sub-agent does. It's a black box within a black box." He also recommended tmux instead of background bash. — [Mario's blog post](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- The README still says pi "skips features like sub-agents and plan mode. Ask Pi to build what you want, or install a package." — [README](https://github.com/earendil-works/pi/blob/main/README.md)
- The example `subagent` extension runs each subagent "in a separate `pi` process". It supports parallel streaming, per-agent usage and cost, and Ctrl+C propagation. It ships agent definitions (scout, planner, reviewer, worker) and chained workflow prompts such as scout→planner→worker. There are also `plan-mode`, `handoff.ts`, `git-checkpoint.ts`, and `dirty-repo-guard.ts` examples. — [subagent example README](https://github.com/earendil-works/pi/tree/main/packages/coding-agent/examples/extensions/subagent)
- pi-durable is marked "Experimental. The API changes without notice". It provides:
  - A Harness with atomic commits ("nothing is shown before its commit is stored") and crash resume.
  - Conversations, immutable Entries, typed Documents (`pi.live`, `pi.inbox`, `pi.usage`, `pi.agent`), and durable checkpointed Tasks.
  - `submit` with `whenBusy: steer | follow-up | reject`.
  - Subagents as conversations owned by a task. Aborts cascade, `{background:true}` tasks survive the parent's abort, and example 23 implements persistent background subagents that report back as follow-ups.
  - Child tasks with `allSettled` or `failFast`.
  - `harness.taskGraph()` for "a task panel".
  - `viewState()`/`watch()` for UIs. Late joiners start from the current view and "nothing is replayed". A slow watcher keeps at most 100 frames before they collapse into one full-state frame.
  - Usage and cost per provider/model.
  - Storage in memory, SQLite (WAL), or JSONL. "One process owns a storage at a time."
  
  — [pi-durable README](https://github.com/earendil-works/pi/blob/main/packages/durable/README.md)
- pi-server (experimental) routes clients to durable Sessions with multiple presentation attachments. It is built on chord and the CBOR pi-protocol. — [pi-server README](https://github.com/earendil-works/pi/blob/main/packages/server/README.md); [chord README](https://github.com/earendil-works/pi/blob/main/packages/chord/README.md)

### Inferences
- Earendil appears to be building a daemon/server version of pi that overlaps heavily with kitchen's planned architecture: durable log, SQLite, task graph, multi-client attachments, and a subagent ownership model. This is both an opportunity (reuse or learn from it) and a risk (competing with, or depending on, an experimental API from a company with commercial plans).
- pi-durable's "nothing is replayed" watch semantics differ from kitchen's seq-based replay. Kitchen would still need its own event log, or would need to fetch snapshots on reconnect.

### Gaps
- When pi-durable, pi-server, and chord first appeared, and their roadmap and stability, could not be determined (shallow clone). Earendil RFCs at rfc.earendil.com/keyword/pi may cover this but were not read.

## Mario Zechner's design writings

### Takeaway
The main primary source is "What I learned building an opinionated and minimal coding agent" (mariozechner.at, 2025-11-30). It argues for a minimal, observable, stable harness and against Claude Code's feature creep and hidden context injection.

### Cited Findings
- Claude Code "has turned into a spaceship with 80% of functionality I have no use for". Its prompts and tools change often, "which breaks my workflows", and the UI flickers. Mario wants to "inspect every aspect of my interactions with the model". He criticizes context injected "behind your back". — [Mario's blog post](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- He argues frontier models are "RL-trained up the wazoo" and need little prompt. Four tools suffice. On YOLO: with read plus network tools "you're playing whack-a-mole". No to-dos ("confuse models more than they help"). No plan mode (use files). No background bash (use tmux). No MCP and no sub-agents (as of then). He cites Terminal-Bench 2.0 results showing pi competitive with other harnesses, and Terminus 2 (raw tmux) doing respectably. Guiding rule: "if I don't need it, it won't be built". — [Mario's blog post](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- pi-tui uses differential rendering with synchronized output and keeps the terminal's native scrollback. Note that 1.0.0 changed the default to fullscreen TUI. — [Mario's blog post](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/); [CHANGELOG 1.0.0](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/CHANGELOG.md)

### Inferences
- Since the acquisition the product has moved away from strict minimalism: MCP, codemode, fullscreen TUI, durable runtime, and server. The core-plus-extension philosophy remains.

### Gaps
- Mario's April 2026 "I've sold out" post, Armin's "Mario and Earendil" post, and HN threads were not read directly.

## Fit for kitchen

### Takeaway
Best fit: use `pi-ai` (and probably `pi-agent-core`) as the in-process library under kitchen's Bun daemon, and keep kitchen's own SQLite event log, worktree management, and Manager logic. Embedding `pi --mode rpc` per worktree is a viable shortcut for "full coding agent" workers. Watch pi-durable/pi-server, which are experimental and solve overlapping problems. The main risks are churn and dependence on Earendil's direction, more than a single-maintainer bus factor.

### Cited Findings
- pi-ai is the cleanest reusable layer. Kitchen would get:
  - About 35 providers.
  - OAuth with a cross-process refresh lock.
  - A pluggable CredentialStore.
  - Cross-provider handoff.
  - Typed streaming events.
  - Tree-shaking.
  - A faux provider for tests.
  
  — [pi-ai README](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md)
- agent-core gives steer, followUp, and abort, plus an event stream suited to persistence. — [agent-core README](https://github.com/earendil-works/pi/blob/main/packages/agent/README.md)
- For full-coding-agent workers, RPC mode is documented for "process isolation, IDEs, and custom user interfaces". OpenClaw shows production use of in-process embedding through the SDK. — [rpc.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/rpc.md); [OpenClaw docs](https://docs.openclaw.ai/pi)
- There is no permission system, so kitchen would supply isolation itself through worktrees plus containers, Gondolin, or other means. — [README](https://github.com/earendil-works/pi/blob/main/README.md)

### Inferences
- **Recommended layering for kitchen:**
  - The kitchen daemon uses pi-ai for model I/O.
  - It uses either agent-core `Agent` per worker or the coding-agent SDK `createAgentSession()` per worktree. The SDK brings pi's read/bash/edit/write tools, compaction, and extensions in-process.
  - Kitchen maps pi events to its seq-numbered SQLite log.
  - The Kitchen Manager is itself an Agent whose tools are spawn_worker, steer_worker, and query_status.
  - RPC subprocess workers are better when kitchen needs crash isolation per worker or mixed runtimes.
- **Bun compatibility:** pi targets Node 22.19+ and has Bun install smoke tests. Plan to verify the coding-agent SDK under Bun, particularly its PTY/bash and TUI dependencies, before committing.
- **Risks:**
  1. Very high release velocity: 1.0 was 2 days old at research time, and the API has had frequent renames and a scope change. Pin and vendor.
  2. Earendil commercial strategy: Fair Source and proprietary layers may sit next to the MIT core, and future durable/server features may end up in commercial products. However, the MIT core is forkable.
  3. Contribution gate: outside PRs are auto-closed by default, which makes upstreaming fixes harder.
  4. Anthropic subscription OAuth is contrary to Anthropic's terms for third-party harnesses.
  5. pi-durable and pi-server overlap with kitchen's planned daemon. They might replace much of kitchen, or kitchen might end up competing with them.
- **Bus factor:** improved. There is now a funded company, Armin Ronacher is the top recent committer, and several other regular committers exist. That is better than the Aug–Mar single-maintainer era.

### Gaps
- No hands-on test was run: Bun compatibility, running multiple concurrent SDK sessions in one process, and RPC schema stability.
- Earendil's commercial product boundaries beyond RFC 0015 are unknown.
