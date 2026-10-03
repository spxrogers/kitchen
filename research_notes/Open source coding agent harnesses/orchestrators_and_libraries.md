# Multi-agent coding orchestrators (prior art) and paved-road libraries for a custom TS harness (as of Oct 2026)

Scope note: Research done 2026-10-03 with ~18 tool calls. Items marked **[training-data, unverified]** come from the researcher's background knowledge (cutoff ~mid-2026) and were not re-verified against a live source in this pass. Treat them as leads, not facts.

## Part A: Which orchestration products and projects are closest to kitchen, and what gap is left?

### Takeaway
The "dashboard of parallel worktree agents" layer is now crowded and mostly commoditized: Claude Code Agent View (May 2026), Codex app, Conductor, Sculptor, vibe-kanban, claude-squad, Emdash, Agent Orchestrator, GitHub Agent HQ / Mission Control, Antigravity Manager view. Two things are still rare: (1) a **conversational manager agent the user talks to**, which dispatches workers *across providers and harnesses*, and (2) a **durable, replayable, cross-harness event log and attention queue** exposed as an API. Claude Code's lead/teammate model is Claude-only, experimental, one team per session, and cannot resume in-process teammates. Most third-party orchestrators are human-driven boards with no manager agent. A few small OSS projects (Vigil, Agent Orchestrator, Bernstein) are starting to fill the manager-agent slot.

### Cited Findings

**Claude Code: Agent View, background sessions, agent teams, subagents (Anthropic, closed product)**
- Agent View launched May 11, 2026 as a Research Preview. You open it with `claude agents` or the left arrow from any session. It is one screen for background sessions with three states: needs your input, working, done. You can peek at the last turn and answer inline, and markers show which sessions produced PRs. Available on Pro/Max/Team/Enterprise/API plans — [Anthropic blog](https://claude.com/blog/agent-view-in-claude-code)
- `/bg` sends a session to the background and `claude --bg [task]` launches one there. Shipped in v2.1.139–v2.1.142 (week 20, May 11–15, 2026) — [Claude Code what's new w20](https://code.claude.com/docs/en/whats-new/2026-w20)
- "Each session is its own Claude Code process, hosted by a per-user supervisor. They keep running with no terminal attached" — [dsebastien.net](https://dsebastien.net/claude-code-agent-view-one-screen-for-every-background-session/) (secondary source)
- v2.1.196 (late June 2026): long-running commands persist across session process stops, restarts and updates. Workers stopped by a daemon restart resume automatically the next time agent view opens — [Classmethod changelog summary](https://dev.classmethod.jp/en/articles/20260630-cc-updates-v2-1-196/) (secondary source)
- Agent teams are **experimental and off by default** (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`). A lead session spawns teammates, each a full Claude Code instance with its own context. They share a task list (pending / in progress / completed, with dependencies and file-lock claiming) and a mailbox (JSON files at `~/.claude/teams/{team}/inboxes/{agent}.json`). The user can talk to any teammate directly — [Claude Code docs: agent teams](https://code.claude.com/docs/en/agent-teams)
- Attention model in teams:
  - Idle notifications go automatically to the lead with the teammate's final answer, and failures carry the error text.
  - Teammate permission prompts "appear in the lead session". Plan approvals are auto-granted by the lead.
  - Hooks `TeammateIdle`, `TaskCreated` and `TaskCompleted` can enforce quality gates (exit code 2 sends feedback).
  - Source: [agent teams docs](https://code.claude.com/docs/en/agent-teams)
- Agent team limitations listed in the docs:
  - `/resume` and `/rewind` do not restore in-process teammates.
  - Task status can lag.
  - One team per session, and no nested teams.
  - The lead is fixed for its lifetime.
  - Permission modes are set at spawn.
  - Teams don't spawn in `-p`/Agent SDK (non-interactive) sessions.
  - Docs advise "Avoid file conflicts… each teammate owns a different set of files". Worktrees are a separate manual option ("Git worktrees let you run multiple Claude Code sessions yourself without automated team coordination").
  - Source: [agent teams docs](https://code.claude.com/docs/en/agent-teams)
- Teammate models come from the spawn prompt, the subagent definition's `model`, then `CLAUDE_CODE_SUBAGENT_MODEL`, then the lead's model. All are Claude models; an org `availableModels` allowlist is enforced. Nothing in the docs supports non-Anthropic models — [agent teams docs](https://code.claude.com/docs/en/agent-teams)

**OpenAI Codex app**
- The Codex app has built-in worktrees for parallel agents and an inline diff reviewer where you stage or revert chunks. Git allows only one worktree per branch, which causes branch-lock friction. The "Overwrite local" apply skips .gitignored files — [Verdent guide](https://www.verdent.ai/guides/codex-app-worktrees-explained) (secondary source)
- **[training-data, unverified]** The Codex desktop app (macOS) launched in early 2026 as a multi-agent "command center" with per-thread worktrees, skills and automations. Codex CLI core is Rust and Apache-2.0. The Codex app itself is closed source.

**GitHub Agent HQ / Mission Control (closed, SaaS)**
- Announced Oct 28, 2025 at Universe. Mission Control is "a single command center to assign, steer, and track the work of multiple agents". It works across github.com, VS Code, CLI and mobile, and drives Copilot coding agent plus third-party agents (Anthropic Claude, OpenAI Codex, Google, Cognition, xAI) through a paid Copilot subscription. It also has a governance control plane and a metrics dashboard — [GitHub blog](https://github.blog/news-insights/product-news/welcome-home-agents/); [Visual Studio Magazine](https://visualstudiomagazine.com/Articles/2025/10/28/GitHub-Introduces-Agent-HQ-to-Orchestrate-Any-Agent-Any-Way-You-Work.aspx)
- Inference: this is cloud and PR-centric, with GitHub-hosted runners rather than local worktrees, and there is no conversational manager agent.

**Google Antigravity / Jules**
- Antigravity is Google's agentic IDE. It has an editor view plus a "manager view" that acts as a control center for orchestrating multiple agents in parallel across workspaces — [Wikipedia: Google Antigravity](https://en.wikipedia.org/wiki/Google_Antigravity); [crystl.dev comparison](https://crystl.dev/blog/best-ai-agent-orchestration-tools/)
- Jules is "an async GitHub task runner with a usable free tier" — [crystl.dev](https://crystl.dev/blog/best-ai-agent-orchestration-tools/)
- Google's "Conductor" (a spec-driven development extension, distinct from conductor.build) now supports Antigravity — [Google Developers Blog](https://developers.googleblog.com/evolving-spec-driven-development-conductor-now-supports-antigravity/)

**Conductor (conductor.build, Melty Labs; closed source, macOS)**
- Starts several Claude Code, Codex or Cursor agents, each in an isolated git-worktree workspace, with one window to watch them, read diffs and ship. Supports routing to Cursor and Antigravity agents — [crystl.dev](https://crystl.dev/blog/best-ai-agent-orchestration-tools/) (secondary source; vendor details not verified at conductor.build)
- It is a human-driven dashboard, and no sources describe a manager agent.

**Sculptor (Imbue)**
- A desktop app for parallel coding agents. Each agent gets its own git worktree, branch and diff view, and everything is reviewed and merged from one window. It ships workflow skills (spec writing, mock generation, TDD bug fixing) — [Imbue product page](https://imbue.com/product/sculptor)
- Described as MIT-licensed with Apple Silicon and Linux builds — [crystl.dev](https://crystl.dev/blog/best-ai-agent-orchestration-tools/); [rywalker.com](https://rywalker.com/research/sculptor). Note: earlier (2025) Sculptor used containers and was not OSS **[training-data, unverified]**. The license change should be checked on the repo.

**vibe-kanban (BloopAI)**
- Apache-2.0 web kanban that orchestrates 10+ agents: Claude Code, Codex, Gemini CLI, Copilot, Amp, Cursor, OpenCode, Droid, Qwen Code. It uses per-workspace worktrees, and an MCP server lets agents decompose tasks into cards — [Augment Code list (Apr 27, 2026)](https://www.augmentcode.com/tools/open-source-agent-orchestrators)
- Each card gets a branch and a worktree, and the selected agent is launched with the card's prompt. "The original company shut down in April 2026"; the repo is now community-maintained — [VirtusLab](https://virtuslab.com/blog/ai/vibe-kanban/)

**claude-squad**
- AGPL-3.0 TUI that runs tmux sessions plus worktrees. Supports Claude Code, Codex, Aider, Gemini, OpenCode and Amp. It is a pure human-in-the-loop session manager with no manager agent — [Augment list](https://www.augmentcode.com/tools/open-source-agent-orchestrators)

**Crystal → Nimbalyst**
- Nimbalyst is listed as the "successor to Crystal". It is a desktop app for Claude Code and Codex with per-session worktrees — [Augment list](https://www.augmentcode.com/tools/open-source-agent-orchestrators). Crystal (stravu) was MIT **[training-data, unverified]**.

**Other OSS orchestrators (Augment survey, Apr 27, 2026)** — [Augment list](https://www.augmentcode.com/tools/open-source-agent-orchestrators)
- **Agent Orchestrator (ComposioHQ, now Untrivial-ai)**: Apache-2.0 desktop app and daemon supporting 26 harnesses. Each git-backed session gets its own worktree, branch and PR, and "milestone gates" retry CI automatically — [GitHub](https://github.com/ComposioHQ/agent-orchestrator)
- **Emdash**: Apache-2.0 (YC W26) Electron app supporting 34 CLI agents. Has ticket intake and runs worktrees locally or over SSH.
- **Bernstein**: Apache-2.0 TUI plus web inspector with 49 adapters, deterministic scheduling and merge gates, with a "Janitor" doing verification.
- **Baton**: MIT. Polls GitHub Issues and runs Claude Code in per-issue worktrees.
- **Microsoft Conductor**: MIT. YAML workflows for Copilot and Claude.
- **Agent Kanban**: ELv2. A VS Code extension for Copilot.

**Projects with an actual manager agent** — [search results summarizing awesome-agent-orchestrators et al.](https://github.com/andyrewlee/awesome-agent-orchestrators)
- **Vigil**: a native macOS terminal where "a manager agent runs the tree — spawning workers and sub-managers". Supports Claude Code, Codex and OpenCode.
- **Paseo**: a self-hosted daemon running Claude Code, Codex, Copilot, OpenCode and Pi in parallel.
- **Orca**: Codex, Claude Code, OpenCode and Pi side by side, one worktree each.
- **Crew** ([pikehouse/crew](https://github.com/pikehouse/crew)) and **claude-team** (Martian Engineering MCP server): both Claude-Code-only.
- These are snippet-level findings and were not individually verified.

**Ecosystem catalogs:** [awesome-agent-orchestrators](https://github.com/andyrewlee/awesome-agent-orchestrators); [awesome-cli-coding-agents](https://github.com/bradagi/awesome-cli-coding-agents); [openorchestrators.org](https://openorchestrators.org/)

**Not researched in this pass (gaps):** Amp, Cursor background agents and multi-agent window, OpenHands multi-agent, uzi. See Gaps.

### Inferences
- **Commoditized:** worktree-per-agent, diff review, PR links, and a needs-input / working / done status list. Every serious tool has these. Kitchen should not compete on the dashboard alone.
- **Claude Code's design is the closest analogue to kitchen**: a per-user supervisor daemon, background sessions, an attention-first view, plus a lead/teammate team with a mailbox and task list. Its gaps relative to kitchen:
  - Single provider (Claude only).
  - Agent View is a session list, not a conversational manager. Agent teams are a manager, but experimental, scoped to one session, not resumable, and not worktree-isolated by default.
  - No external API or event log for other UIs.
- **Multi-harness dashboards** (vibe-kanban, Emdash, Agent Orchestrator, claude-squad) drive many CLIs, but the human is the dispatcher. Kitchen's differentiator would be the "Kitchen Manager" LLM that:
  - turns a conversation into jobs,
  - picks the model or provider per job,
  - monitors workers and triages "what needs me" into a ranked queue,
  - persists everything in a replayable seq-based log (HTTP+WS) that other clients can subscribe to.
- **Risk:** the funded-company exit rate is high (Bloop shut down vibe-kanban's company), while first-party products (Anthropic, OpenAI, GitHub, Google) are absorbing the dashboard layer. A personal/self-hosted, provider-neutral niche is more defensible than a product race.
- **Practical design hint:** let kitchen drive *existing harnesses* (Claude Code via Agent SDK/headless, Codex via `codex exec`/app-server or ACP, Pi, OpenCode) as worker backends, alongside or instead of its own loop. This is what Agent Orchestrator, Emdash and AI SDK 7 HarnessAgent all do.

### Gaps
- Not verified: Amp (Sourcegraph) subagents/"oracle", Cursor 2.x/3 multi-agent window and background agents, OpenHands multi-agent/agent-server, uzi, Codex app details from primary docs, conductor.build's own feature page, and Sculptor's current license on its repo.
- Notification models (desktop/push/mobile) for most third-party tools were not documented in the sources found.
- Star counts and activity levels were not checked.

## Part B: Which paved-road libraries can a custom multi-provider coding harness be built on?

### Takeaway
For a Bun/TS harness, the realistic shortlist is:
- **Vercel AI SDK (v7, June 2026)**: the broadest provider layer plus `ToolLoopAgent`, approvals, a sandbox abstraction, a TUI, and the new `HarnessAgent` that wraps Claude Code, Codex and Pi.
- **pi-ai / pi-agent-core**: a minimal, coding-agent-specific design with cross-provider context handoff.

Mastra, LangGraph.js and the OpenAI Agents SDK add orchestration or opinions that kitchen's own daemon would duplicate. The Claude Agent SDK is the strongest *Claude worker backend* but is governed by Anthropic Commercial Terms and is Claude-centric. Whichever library you pick, you still own the coding-specific layer: edit tool semantics, the bash sandbox, permissions, compaction, cache-aware prompt layout, and session persistence.

### Cited Findings

**Vercel AI SDK**
- AI SDK 6 introduced `ToolLoopAgent` (define model, instructions and tools once and reuse), human-in-the-loop tool approval (`needsApproval`), stable structured output, and DevTools — [Vercel blog: AI SDK 6](https://vercel.com/blog/ai-sdk-6)
- AI SDK 7 shipped June 25, 2026 — [Vercel changelog](https://vercel.com/changelog/ai-sdk-7). It contains:
  - `HarnessAgent`, which runs external harnesses through the standard `Agent` interface with `generate`/`stream` results.
  - `WorkflowAgent`, a durable long-running agent whose state is persisted between steps.
  - `ToolLoopAgent` with runtime context and approval policies (require, auto-approve, auto-deny, or a typed function).
  - Sandbox support: command execution, streaming output, working dirs, env, abort signals, and step-level sandbox overrides.
  - `@ai-sdk/tui`.
  - Total, per-step, per-chunk and per-tool timeouts.
  - MCP Apps support.
  - `@ai-sdk/otel` telemetry.
  - Requirements: Node 22 minimum, ESM only.
- AI SDK harnesses wrap "a complete agent runtime, such as Claude Code, Codex, or Pi", which brings "workspace access, built-in coding tools, native session state, compaction, permission flows". They are **experimental** ("Expect breaking changes"). Sessions carry runtime, sandbox, cwd, native history and pending approvals, and resume via `session.detach()`/`stop()` with a stable `sessionId`. Harness-specific events such as file changes and compaction surface as tool parts — [AI SDK docs: harnesses](https://ai-sdk.dev/docs/ai-sdk-harnesses)
- Reported usage is over 16M weekly downloads, and v7 is pitched as an "agent platform" — [search summary of Vercel changelog/runtimewire](https://runtimewire.com/article/vercel-ai-sdk-7-adds-harnessagent-for-coding-agent-harnesses)
- **[training-data, unverified]** License is Apache-2.0. It has the widest first-party provider list (OpenAI, Anthropic, Google, xAI, Mistral, Bedrock, Azure, Groq, OpenAI-compatible, and others). It generally works on Bun because it is plain ESM fetch-based code, but Bun is not officially listed as supported. Anthropic prompt caching is exposed via `providerOptions.anthropic.cacheControl`. There is no built-in compaction.

**pi-ai / pi-agent-core (Mario Zechner, MIT)** (brief, since this is covered in other notes)
- Thesis: four tools (read, write, edit, bash), a system prompt under 1,000 tokens, an exact-match `oldText` → `newText` edit, and no MCP (examples given: Playwright MCP is 21 tools and 13.7k tokens; Chrome DevTools MCP is 26 tools and 18k tokens, "7-9% of your context window"). No subagents ("black box within a black box"), and the TUI uses differential rendering — [Zechner blog, Nov 30 2025](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- Provider abstraction is hard:
  - Cerebras, xAI, Mistral and Chutes reject the `store` field.
  - Reasoning arrives in different fields (`reasoning_content` vs `reasoning`).
  - Some providers report token usage at the start of the SSE stream, others at the end.
  - pi-ai's fix for cross-provider handoff converts Anthropic thinking to `<thinking>` blocks when switching to OpenAI.
  - Source: [Zechner blog](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- pi with Claude Opus 4.5 was competitive on Terminal-Bench 2.0, and the bare-tmux "Terminus 2" agent "is holding its own" — [Zechner blog](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- Pi is one of the three harnesses AI SDK 7 wraps — [AI SDK harnesses](https://ai-sdk.dev/docs/ai-sdk-harnesses)

**Claude Agent SDK**
- Use is governed by Anthropic's Commercial Terms of Service, including in products you offer to your own customers. "The Python SDK is MIT-licensed; the TypeScript SDK is publicly available but its use is governed by Anthropic Commercial Terms rather than MIT" — [Claude Agent SDK docs](https://platform.claude.com/docs/en/agent-sdk/overview); [futureagi summary](https://futureagi.com/blog/what-is-claude-agent-sdk-2026/) (the MIT-vs-Commercial split is from a secondary source)
- It is the Claude Code runtime as a library (tools, compaction, permissions, hooks, subagents, MCP). Agent teams do not spawn in Agent SDK / `-p` sessions — [agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **[training-data, unverified]** Models are Claude-only: Anthropic API, Bedrock, Vertex, Foundry, or via `ANTHROPIC_BASE_URL`. Proxies such as LiteLLM/OpenRouter Anthropic-compatible endpoints can point it at other models, but this is unsupported and fragile. The TS package spawns the bundled Claude Code CLI as a subprocess.
- I found no primary source on OpenRouter or third-party model use.

**Mastra**
- TS-native, from the Gatsby team. 1.0 shipped January 2026, with about 22k stars and about 300k weekly npm downloads at that time. It bundles agents, workflows, memory and evals and is opinionated — [alexcloudstar comparison](https://www.alexcloudstar.com/blog/ai-agent-frameworks-comparison-2026) (secondary source)
- **[training-data, unverified]** Built on AI SDK model providers. Core is Apache-2.0 with some enterprise ("ee") directories under a separate license.

**LangGraph.js**
- A "low-level orchestration framework and runtime for stateful, multi-actor agents" with a state graph, checkpointing, streaming and HITL interrupts. It has the steepest learning curve — [alexcloudstar](https://www.alexcloudstar.com/blog/ai-agent-frameworks-comparison-2026)
- **[training-data, unverified]** MIT. Providers via LangChain.js integrations. The Python version (LangGraph / "Deep Agents") is more mature than the JS one.

**OpenAI Agents SDK (JS)**
- "A thin, official-blessed wrapper around the agent loop", best when you are heavily on OpenAI (Responses API, Realtime, hosted tools) — [alexcloudstar](https://www.alexcloudstar.com/blog/ai-agent-frameworks-comparison-2026)
- **[training-data, unverified]** MIT. Provides handoffs, guardrails, tracing and sessions. Non-OpenAI models go through an AI SDK adapter (`@openai/agents-extensions`).

**ACP (Agent Client Protocol)**
- Created by Zed in August 2025. JSON-RPC 2.0 over stdio between agents and editors. Claude Code and Codex participate through adapters; `codex-acp` is open-sourced. In January 2026 Zed and JetBrains launched the ACP Registry, and about 50 agents implemented the spec as of June 2026 — [Morph explainer](https://www.morphllm.com/agent-client-protocol); [Zed ACP](https://zed.dev/acp); [Zed: Codex is live](https://zed.dev/blog/codex-is-live-in-zed)
- Inference: ACP is the most practical *uniform worker-control protocol* for kitchen. A kitchen worker = spawn an ACP agent (claude-code-acp, codex-acp, gemini, opencode, pi...) in a worktree, then map ACP `session/update` and permission requests into kitchen's event log and attention queue.

**MCP SDK**
- **[training-data, unverified]** `@modelcontextprotocol/sdk` (MIT) is the standard. Zechner's context-cost critique of MCP tool sprawl applies — [Zechner](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)

**Provider routers and registries** (no primary sources fetched this pass)
- **[training-data, unverified]** **OpenRouter**: hosted, OpenAI-compatible, many providers, passes through some Anthropic cache_control.
- **[training-data, unverified]** **LiteLLM**: MIT Python proxy/SDK that can expose OpenAI-format and Anthropic-format endpoints. Its Anthropic `/v1/messages` passthrough lets Claude Code or the Claude Agent SDK hit other models.
- **[training-data, unverified]** **Vercel AI Gateway**: hosted, works natively with AI SDK model strings, BYOK.
- **[training-data, unverified]** **models.dev**: an MIT open registry of model metadata (context limits, pricing, capabilities) from the SST/opencode team, used by OpenCode.
- Inference: for kitchen, direct provider SDKs or AI SDK providers plus models.dev metadata avoid a proxy hop. OpenRouter is useful as a catch-all job backend.

**Effect AI:** no sources found in this pass (see Gaps).

### Inferences
- **Build-on-Codex-core vs roll-your-own.** Codex core gives a hardened sandbox (seatbelt/landlock), apply_patch, and compaction, but it is Rust, OpenAI-centric in prompting, and has its own state model. For kitchen, the *orchestrator* is the product. Wrapping harnesses as workers (ACP or AI SDK HarnessAgent) plus one in-house lightweight loop for the Kitchen Manager seems lower-risk than forking a harness.
- **Suggested split:**
  - The Kitchen Manager runs on AI SDK `ToolLoopAgent` or pi-agent-core. It needs few tools: dispatch_job, read_job_log, message_worker, summarize, raise_attention.
  - Workers are existing harnesses (Claude Code / Agent SDK, Codex, Pi, OpenCode) via ACP or headless modes, plus optionally kitchen's own pi-style worker for "any model" jobs.
- **Bun:** AI SDK and pi-ai are fetch/ESM-based and very likely work. The Claude Agent SDK TS spawns a CLI subprocess, which should work. AI SDK 7 officially states Node 22+. Test Bun early.
- **What you still build on any library:**
  - edit tool semantics (exact-match replace, apply_patch, or both, per model family),
  - a bash tool with timeouts and output truncation,
  - the sandbox/permission policy,
  - compaction/summarization,
  - cache-stable prompt layout,
  - session persistence/resume,
  - cross-provider message normalization (thinking blocks, tool-call IDs, usage).

## What does it realistically take to build a competent coding agent loop in 2026?

### Takeaway
Practitioners converge on a small core:
- 4–6 tools (read, write, exact-match edit, bash, optionally grep/glob/ls),
- a short system prompt,
- a stable cached prefix,
- truncation of tool output plus compaction of old turns,
- structured session memory,
- optional bounded subagents.

The hard parts are provider normalization, context management and sandbox/permissions, not the loop itself.

### Cited Findings
- Pi: read/write/edit/bash, a system prompt under 1k tokens, and exact-whitespace `oldText` matching for edits. It reached competitive Terminal-Bench 2.0 results with Opus 4.5 — [Zechner](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- Raschka's six components (Apr 4, 2026) — [Raschka, Ahead of AI](https://magazine.sebastianraschka.com/p/components-of-a-coding-agent):
  1. live repo context (branch, layout, docs, status),
  2. a stable cached prompt prefix (instructions, tool descriptions, workspace summary) with only the tail changing,
  3. validated structured tools,
  4. context reduction (clip long outputs, dedupe old file reads, compress older transcript),
  5. two-layer session memory (distilled working memory plus full transcript as JSON for resume),
  6. bounded subagents.
- Claude Code itself: in-process teammates/subagents fall outside the main conversation's cache TTL bucket and default to a 5-minute cache, with `subagentPromptCacheTtl: 1h` available at the higher write price. Prompt-cache TTL management matters for multi-agent cost — [agent teams docs](https://code.claude.com/docs/en/agent-teams)
- Multi-agent costs scale linearly per worker. Docs recommend 3–5 teammates and 5–6 tasks each — [agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **[training-data, unverified]** Further practitioner sources worth citing in the report:
  - Thorsten Ball, "How to Build an Agent" (ampcode.com, Apr 2025): a working agent in about 400 lines with read_file, list_files and edit_file.
  - Anthropic, "Effective context engineering for AI agents" (Sep 2025): compaction, note-taking, sub-agent context isolation.
  - Aider's edit-format benchmarks (diff vs whole vs udiff, per model).
  - OpenAI's `apply_patch` format, which GPT-5/Codex models are trained on. Edit format should therefore vary by model family.

### Inferences
- Minimum viable worker loop for kitchen:
  - stream → tool calls → execute (with permission check) → append results → repeat until no tool calls or a budget is hit;
  - tools: read (with line ranges), write, edit (exact-match; offer apply_patch for OpenAI models), bash (timeout, truncated output, cwd pinned to the worktree), grep/glob;
  - a short system prompt plus AGENTS.md/CLAUDE.md injection;
  - Anthropic cache breakpoints on system and tools plus the rolling last message;
  - compaction at about 70–85% of context via a summarize-and-restart that keeps the task, plan and files touched.
  - Effort estimate: days for a loop, weeks for robust cross-provider normalization, sandboxing and resume.
- Kitchen's append-only SQLite event log is a natural home for the "full transcript" layer. The distilled working-memory layer is what the Manager reads to triage attention.

### Gaps
- No primary sources fetched this pass for Thorsten Ball, Anthropic's context-engineering post, Aider edit-format benchmarks, or OpenAI apply_patch guidance. They are listed from background knowledge.
- No quantitative 2026 data comparing edit formats across current models (e.g. GPT-5.x, Claude 4.6+/5, Gemini 3) was found.
- Effect AI (`@effect/ai`) status, license and provider coverage were not researched.
- License, Bun compatibility, and exact provider counts for Mastra, LangGraph.js, OpenAI Agents JS, models.dev, LiteLLM, OpenRouter and Vercel AI Gateway were not verified against primary sources this pass.
