# Cloud sessions as a kitchen

**Last updated:** 2026-11-20 · **Experiments run:** 2026-11-19 to 2026-11-20

Kitchen was shelved when Claude Code Agent View shipped its attention UX for local sessions. This page records one evening of hands-on experiments with the other half: Claude Code **cloud sessions** (claude.ai/code). The question was whether a single long-running cloud session can act as the Kitchen Manager: dispatch workers, watch them, say what needs you, and keep a durable queue, with nothing to build or host.

Short answer: for personal, cloud-first work, mostly yes. The platform already provides most of what spec §5 to §12 planned to build. The gaps are approvals, live state inside a web page, and anything that runs on a local machine.

Everything below was observed directly unless marked otherwise. The experiments ran against a private app repository and a handful of personal projects. Their names, IDs, numbers and data are left out.

Related: [`landscape.md`](landscape.md) (open-source harnesses and orchestrators) and the shelved [v1 spec](superpowers/specs/2026-04-29-kitchen-design.md).

---

## TL;DR

1. **A cloud session can run other cloud sessions.** It can create them, give them tasks, message them, read their transcripts, interrupt them and archive them. Workers report back with a message. That covers kitchen's Kitchen Manager MCP server (spec §10.2), including `dispatch_worker`, which the spec had deferred past v1.
2. **The trust boundary sits in the right place.** Workers do harmless work a parent asks for. They refuse destructive or access-widening requests unless the owner says so in that worker's own session. A parent cannot grant permissions, even by writing them into the worker's system prompt.
3. **Spawning nests at least five levels deep**, and a worker at any depth can message the top directly.
4. **The durable queue can be an Artifact page with its own database.** The Kitchen Manager session writes tickets with a tool and the owner clears them on the page. Both directions were verified.
5. **A page can read live data through claude.ai connectors** (Postgres through a Neon connector worked), but not session state. The session connector did not load inside a page, so the Kitchen Manager writes a snapshot instead.
6. **Local machines and work accounts don't fit cleanly.** Cloud-to-laptop messaging needs an explicit owner approval per session, and sessions in another account are invisible. Keep a local tool for that side.

---

## The primitives a cloud session has

Cloud sessions get a set of session-management tools from the cloud runtime (named `claude-code-remote`). They are not in any config file and are not publicly documented. A local CLI session does not have them.

| Tool | What it does | kitchen equivalent |
|---|---|---|
| `list_sessions` | Every session on the account: title, status, `status_bucket` (working / blocked / review_ready / completed / failed), a post-turn summary with `needs_action`, `parent_session_id` | `list_workers()` and the attention snapshot |
| `get_session`, `list_events`, `get_event` | One session's record and its transcript | `get_worker`, `tail_events` |
| `create_session` | New session with an initial prompt, `append_system_prompt`, one `source_url` (repo), `outcome_branch`, permission mode, extra allowed tools | `dispatch_worker` (deferred in v1) |
| `send_message` | Message another session (`now`, `next` or `later`) | `sendInput` |
| `interrupt_session`, `archive_session`, `unarchive_session` | Lifecycle | `interrupt`, `kill`, archive |
| `send_later`, `create_trigger` | One-shot or recurring wake-ups (hourly minimum) | Scheduled briefings (deferred in v1) |
| `add_repo` | Attach another repo to the current session | (none) |

From a local machine the documented equivalents are CLI commands, not tools: `claude --cloud "<task>"` starts a cloud session, and `claude -p "<msg>" --cloud <id>` posts into one. There is no documented way to list or read cloud sessions locally.

## Messaging

- Delivery takes a few seconds. A message to an idle session starts a turn.
- The receiver gets the message wrapped as data from another session, not as input from its owner. It is told not to change permissions or take new destructive actions on that basis alone.
- A reply comes back only if the receiver calls `send_message` itself. Children can address their creator as `@parent`.
- A message from a parent was enough for a child to do ordinary work: run a command and report the output, or do a read-only code review with two subagents and send back a scored report.

## Trust: what a parent can and cannot make a worker do

| Request to a worker | Result |
|---|---|
| Run a harmless command and report back | Done, within minutes |
| Read-only codebase review, report by message | Done |
| "Open a PR deleting all the code", relayed from the owner by the parent | Refused. It came second-hand, broke the repo's PR rules, and improved nothing. Nothing was created. |
| Attach another repo (read-only), relayed | Paused. The worker asked the owner in its own session first. |
| Same, with the owner's approval written into the worker's `append_system_prompt` | The model accepted it. The tool permission check (auto mode) refused it as unauthorized. |
| Same, owner typing in the worker's session | Done |

Three independent layers produced this:
1. **The worker's own judgement.** Approvals count only when the owner types them in that session.
2. **The tool permission check.** It runs separately from the model, and text a parent writes cannot unlock it.
3. **Platform rules.** For example, the cloud-to-laptop gate below.

A parent can shape what a worker does but cannot widen what it may do. Children also can't exceed the parent's permission mode, and `extra_allowed_tools` passes on only tools the parent already has pre-approved.

The permission check is not deterministic. It blocked one harmless report message as "Unsanctioned Communication", then let identical messages through seconds later.

## Spawning depth and scope

A chain where each level spawned the next reached **five levels** without hitting a limit (`depth 1 → 5`). Each new session took about 20 seconds to start.

- **Direct reporting works at any depth.** Levels 2 to 5 all messaged the top session directly, as well as `@parent`.
- **Scope is fixed by the spawn prompt.** Asked to do more by a session further up the tree (not its direct parent), a level-2 worker and a level-4 worker both declined. Their spawn prompts said "do nothing else" or "these are your complete instructions". Each went ahead when the owner said yes in its own session.
- **Spawn prompts are judged, not trusted automatically.** Levels 1 to 4 acted on prompts that said who set the task up and at whose request. Level 5 got a bare prompt ("run this, message that session") and treated it as an unexplained background request until the owner confirmed. That is one example, so it suggests a cause rather than proving one.
- **Subagents work at any depth.** A level-5 session ran an in-session subagent.
- **Repos are not inherited.** A child gets only the repo named at creation, and `create_session` takes one. Repos a parent added later don't flow down, and a worker attaching more needs the owner's approval in that worker.
- **Each session has a fixed startup cost** of about $0.35 to $0.40 for a trivial task. Heavier sessions cost more: a codebase review with two subagents was about $5. Small jobs belong in in-session subagents, and new sessions are for real chunks of work.

## Seeing the tree

- `list_sessions(children: true)` returns direct children only.
- `list_sessions(mine: true)` returns every session on the account, each with `parent_session_id`. That is enough to rebuild the full tree:
  1. List newest-first (results came back sorted by `created_at`, newest first).
  2. Stop at the first row older than the Kitchen Manager, since no descendant can predate it.
  3. Keep rows whose parent chain leads back to it.
- Tag filtering in `list_sessions` is not available from inside a session.

## Archive semantics

| Question | Answer |
|---|---|
| Does archiving a parent archive its children? | No. Children stay active and keep `parent_session_id`. |
| Sidebar | The app nests sessions under a parent that isn't archived. Orphans move to the top level with their subtrees. |
| `@parent` to an archived parent | Fails with `session_inactive`. Not queued, and it doesn't unarchive the parent. |

So "close out X" has to archive a subtree deepest-first, and each worker should also know the Kitchen Manager's session ID as a fallback address.

## Cloud and local

- **Laptop to cloud works.** A local session with Remote Control on messaged a cloud session.
- **Cloud to laptop is off by default.** The send was refused: a cloud session may message a session on its owner's computer only after the owner approves it on a trusted device, which also restricts that cloud session to trusted devices from then on. The reason: a cloud session reads untrusted content, so reaching a real machine is gated.
- **Laptop sessions show up in `list_sessions`** (environment kind `bridge`) while Remote Control is connected.
- **Folders on the laptop can be attached** to a cloud session (shared from the desktop app), turning its commands into commands on the laptop. One folder per session. The machine must stay awake and online. This was not tried end to end.
- **A different account is a hard wall.** A cloud Kitchen Manager on a personal account cannot see sessions in a work account. Keep a local, work-side tool for that.

## Containers

- Each session's container is ephemeral. Local files, including the scratchpad, were gone after the session was archived and unarchived. The conversation survived.
- What lasts is what leaves the container: commits, issues, Artifact pages and their databases, and messages to other sessions.
- A long session's context gets summarized as it grows, so the conversation is a weak backup for exact text.

## The queue: an Artifact page with a database

A published Artifact page can declare a `db` capability: JSON documents in collections, with access rules, live `onSnapshot` updates, and survival across republishes, under 25,000 documents. A session reads and writes the same store with the `ArtifactData` tool.

A spike called **The Pass** tested the loop:
- The Kitchen Manager seeded ten tickets from the account's open sessions (needs you, in flight, waiting, parked).
- The owner reordered two tickets and added one on the page.
- The session read the changes back, with the page-made ticket marked `source: "page"`.

That makes the page the owner's surface and the database the shared state. The Kitchen Manager session becomes replaceable, because no state lives only in its conversation.

Other page capabilities that matter here: `user` (who is viewing), `sample` (the page can ask Claude directly), and `dictation` (voice input in Chromium and the desktop app, not the mobile apps).

## Live data on a page

The `mcp` capability lets a page call the viewer's claude.ai connectors with the viewer's credentials.

| Connector | Result in a page |
|---|---|
| Neon (Postgres), `run_sql` | Worked. An aggregate query returned 12 rows in about 2.2 seconds and matched the same query run from a session. |
| Claude Code Remote (the session tools), `list_sessions` | Failed with "Couldn't load this connector". |

So a page can show live operational data from anything behind a connector, but session state has to come from the Kitchen Manager writing a snapshot into the page's database, refreshed on each pass or on a schedule.

Caution: such a page runs whatever query its code holds, as the viewer, against anything the connector can reach. Keep these pages to aggregates. A private page is one Share click from being public.

## kitchen's spec against the cloud platform

| Spec | Cloud sessions today |
|---|---|
| Workers in isolated workspaces (§12) | A fresh container per session, with no worktree cleanup |
| Event log + live stream (§7, §8) | Transcripts (`list_events`), with live viewing in the app; no structured event API to subscribe to |
| Kitchen Manager as a worker with an MCP server (§10) | Any cloud session; the tools are built in |
| "Which workers need me?" (§10.5) | `status_bucket` + `needs_action`, ranked by wait time |
| `dispatch_worker` (deferred) | `create_session` |
| Scheduled briefings (deferred) | Scheduled tasks |
| Notifications (deferred) | Push notifications to the phone |
| Drop in on a worker | Open it in the app or on the web |
| Peer messaging (a non-goal) | Exists, with the trust rules above |
| Daemon, TUI, macOS app | Nothing to build |

**Where it falls short of kitchen:**
1. **The Kitchen Manager can't answer a worker's permission prompt.** Kitchen's `sendInput` could. Every approval goes to the owner, in the worker's session.
2. **Claude only.** No Codex or other workers, which was kitchen's first design goal.
3. **No structured, replayable event log.** Transcripts exist, but nothing like spec §7's `seq` subscription.
4. **Local and work-account sessions sit outside it.**

## Design notes for a cloud Kitchen Manager

- **Durable state outside the conversation**, in an Artifact database. The Kitchen Manager is replaceable; the queue is not.
- **Spawn prompts with a standing remit and provenance.** Say who set the task up and why, put the remit in `append_system_prompt`, and leave room for follow-ups ("take follow-up tasks from your parent") rather than a closed list of steps.
- **Two report addresses per worker:** `@parent` and the Kitchen Manager's session ID.
- **One worker per repo**, since repos aren't inherited.
- **Close out deepest-first,** checking each worker pushed its work before archiving it.
- **A permission rule for `send_message`** in the environment, so reports aren't dropped by the classifier.
- **Snapshot session state into the queue's database** on every pass, and on an hourly schedule if wanted.
- **One queue for real approvals,** so the remaining friction reaches the owner in batches.

## Open questions

- The real depth limit, if any.
- Whether the session connector will load inside Artifact pages.
- Whether spawn-prompt trust is consistent enough to rely on.
- Cost at the scale of several always-on leads with their own workers.
- Whether attached laptop folders behave well enough to cover the local side.
