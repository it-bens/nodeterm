# Subagent cards, loop cards and Branch

How the canvas draws subagents (ephemeral `SubagentNode` cards), recurring `/loop` / `/schedule` / `/cron` work (`LoopNode`), and the Branch conversation action. Covers the normalized subagent events from Claude's native hooks and the older tool-call reconstruction, the codex and grok variants, the live tails, the reload replay, and the removal rules that Eco's hibernation guard depends on. The hook transport and normalizers are described in [hook-model.md](hook-model.md).

## Subagent visualization

Agents in `SUBAGENT_CAPABLE` emit `subagent-start` / `subagent-end` normalized events, which drive a transient `state/agentNodes.ts` store. For Claude they come from Claude's own `SubagentStart` / `SubagentStop` hooks whenever a session sends them (below). The older reconstruction is kept as the fallback and as the source of the task label: `PreToolUse` / `PostToolUse` on tool `Agent` / `Task` correlated by `tool_use_id`, whose PostToolUse on an async launch is only an ack (`status:'async_launched'`), with the real end sniffed from the `<task-notification>` queued into the parent transcript (context tails produce a synthetic `subagent-end` in both shells). Neither the notification's `UserPromptSubmit` nor the `[Subagent hand-back]` one is a `newTurn`, so neither clears the fan-out.

Canvas renders each subagent as an ephemeral `SubagentNode` (display-only card: type, task, working/done) connected by an edge to its parent agent node. These nodes and edges live outside the React Flow `nodes` state (merged only at the `<ReactFlow>` prop), so they are never persisted (`flowToNodeStates`) nor in undo/dirty. Subagents share the parent's process: no PTY. Each card shows duration, tokens and tool-uses, and expands (click) to a live transcript: `core/subagent-tail.ts` tails the subagent's own transcript file (`<…>/<sessionId>/subagents/agent-<id>.jsonl`; for a native card at the path derived from the parent's transcript and the `agent_id`, `claudeSubagentTranscriptPath`; for a tool-path card matched by `tool_use_id` via the sibling `.meta.json`), read-only, formats each line (assistant text, tool calls, results), and streams chunks over `agent:subagent-activity` into the store.

### Two different clears, and the difference is load-bearing (issue #547)

- The removal paths (node delete, project delete, the cross-project close, the orphan-session kill, `SessionEnd`) call `clearForParent`, which drops everything: a node that is gone has no work left to represent.
- A new turn calls `clearFinishedForParent`, which drops only `state === 'done'`. "The previous fan-out is stale by definition" is true of a finished card and false of a working one. Claude launches subagents async, so "waiting for N background agents to finish" is exactly the state in which the next prompt gets typed, and nothing rehydrates `byId` afterwards (`start()` fires only from a live launch event; a running subagent emits no second one). The card was gone for the rest of the run while the agent kept working. The expensive half is not the missing card: Eco's hibernation guard derives `liveSubagents` from this same store, so the wipe let a parent with live background agents read as idle and get its CLI `/exit`ed.
- Keeping an unfinished card owes a decay: `useAgentNodes.sweepStaleWorking`, on the same 60 s tick and the same `WORKING_STALE_MS` as `agentStatus`'s (imported, never re-chosen: `shared/agents/stale.ts` exists because three surfaces each invented their own timeout). Otherwise a subagent whose end never arrives pins its card, and its parent, forever. It marks the card done rather than deleting it, so a late `finish()` is the no-op it already was and the next turn boundary takes it.
- **A third removal path is opt-in:** `settings.autoHideFinishedSubagentCards` (default off; off reproduces the two paths above exactly) mirrors into the store as `autoHideFinished`, and with it on a card is dropped the moment its subagent reports done, with no turn boundary. Only a done card is ever dropped, so #547's rule and Eco's `liveSubagents` are untouched. The decay is deliberately not one of the paths it gates: `sweepStaleWorking` fires precisely because the end never arrived, which is the opposite of what the setting promises, and it is the one case where a visible card carries the most information (drop it and a fan-out whose subagents died silently leaves nothing on the canvas saying one was ever launched). It costs Eco nothing either way, since an absent card and a done card are the same answer to `liveSubagents`. Renderer only: Desktop and Server Edition identical, Mobile N/A.

## Claude's native subagent hooks

`CLAUDE_HOOK_EVENTS` subscribes `SubagentStart` and `SubagentStop` for every installer (local, managed account dirs, SSH host). Measured on Claude Code 2.1.284 in a throwaway `CLAUDE_CONFIG_DIR` (nine scenarios, print mode and the interactive TUI; fixture `src/shared/agents/__fixtures__/claude/subagent-hook-payloads.json`, pinned by `normalize.claude.subagent-capture.test.ts`). The published npm bundles date them: `SubagentStop` gained `agent_id` and `agent_transcript_path` in 2.0.42, `SubagentStart` first ships in 2.0.43. Facts a refactor must not lose:

1. Both events carry the parent's `session_id` and `transcript_path` (unlike grok, whose stop carries the child's), and `agent_id` (`a` + 16 hex, validated as a token by `isClaudeAgentId` because it becomes a card key and a file name) is the one id they share. The start names the child only by `agent_id` + `agent_type`: no `tool_use_id`, no task text. The stop adds `last_assistant_message` and `agent_transcript_path`.
2. `SubagentStop` is the end of the child's turn and arrives before the `<task-notification>`, sync or async. But a background child that stops while its own child still runs is resumed under the same `agent_id` (a second start, then a second stop), so a native stop does not always mean "finished".
3. Claude fires `SubagentStop` for internal side-agents (prompt suggestions, after nearly every interactive turn) with `agent_type: ""` and no start.
4. A killed child (interrupt) fires no stop.
5. Nested children fire both events through the same subscription and connect flat to the owning node. The tool path never saw them (their `PreToolUse` carries `agent_id` and is filtered), so native hooks are the first time a nested subagent gets a card at all.
6. `Stop` (and `SubagentStop`) carry `background_tasks`: every running background task of the session (async subagents including nested ones, background shells), never a foreground subagent. On `SubagentStop` the finishing child still lists itself, so only the parent `Stop`'s copy is read (`liveBackgroundTaskIds`, closed set of finished statuses, anything else counts as running). Absent through 2.1.112, present by 2.1.266 (not bisected; feature-detected per payload).
7. In interactive auto mode every `PreToolUse(Agent)` of a message fires first, then the children start within 5 ms of each other (the permission classifier sits between; 20 ms gap in print mode, up to about 5 s interactive), each followed about 1 ms later by its async ack whose `tool_response.agentId` names the exact child.
8. The child's `SubagentHandback` tool injects `<agent-message from="…"> [Subagent hand-back] …` into the parent before the task-notification. That is not a genuine turn (`isInjectedSubagentPrompt`, matched on the whole marker).

### How the two paths coexist

One core module, `core/claude-subagent-lifecycle.ts` (`ClaudeSubagentLifecycle`), is fed every normalized event (and the task-notification end) by both shells before any consumer; events it does not act on come back as the same object. A consumer that keys subagents by id must honour `supersedes`.

- **Latch per node+session on the first native start.** Before it, a tool call draws its card immediately (an old CLI, or a session whose hook snapshot predates the upgrade, is byte-for-byte the old stream, pinned over the fixtures with the native events stripped). After it, a tool call is only a pending label and the card appears at the child's own `SubagentStart` (so a denied tool call draws nothing).
- The session's first child is drawn from its tool call and then replaced by its native card (`supersedes`: the renderer store, the host replay and the notch HUD move the card; nothing can know at the tool call that a native start is coming).
- Native cards are keyed by `agent_id`, so start/stop/resume follow the CLI exactly. Only the label is paired, first-in-first-out by type, corrected exactly by the ack (also when the ack overtakes its start, and a call an ack already named is never handed to another child), and for a sync child by its end (`tool_response.agentId`), which takes its call out of the queue and relabels a still-running sibling that guessed it.
- Every turn-end `Stop` / `StopFailure` (never the `idle` rescue, since an Agent call may be waiting on a permission prompt) clears the queue of calls whose child never started, with or without an inventory: 2.0.43 had native hooks long before `background_tasks`, and a denied call's label must not go to the next child.
- A native stop for an id that never started is dropped (side-agents). A later start of a known id re-opens its card. The parent `Stop` inventory, when present, ends a native card it no longer lists (the killed child). A tool card whose child never started is ended at the turn end. A replaced tool card also gets a plain end after the replacing start (for a consumer too old for `supersedes`). Tool-path ends are re-keyed onto the native card (idempotent, and they bring the sync stats the native stop lacks: a late stats-bearing `finish()` fills them on a done card).
- **Tails.** The native start begins the child's tail in the raw listener, which must run before the `ignoreQuestionHook` child-event gate (it ignores every `agent_id`-tagged payload). The lifecycle's `onRelease` ends it (local and remote). A resumed child continues from its remembered offset (`subagent-tail` / `remote-subagent-tail`) instead of re-streaming. A remote child is tailed at its derived host path with no `.meta.json` ssh polling.
- **Eco.** Because a native stop can be a pause (fact 2), the parent `Stop`'s non-empty inventory stamps `backgroundTaskAt` (Canvas), the guard Eco and the bulk restart already read. That is a strictly safer rule than before (it also covers a background shell a subagent launched). Eco safety rides this inventory, not the card alone.
- Both shells are pinned by `hook-verified-parity.test.ts`; the Server Edition also behaviorally over the fixture (`server/agent-status.test.ts`).
- **Cost:** one extra managed-hook process and POST per interactive turn (the side-agent stop).
- **Residuals:** a sync child has no ack, so while it runs a reordered same-type burst can show a sibling's label (never lifecycle) until the first of them ends; a killed child with no later parent `Stop` inventory still waits for the decay.
- Measure a CLI change against real payloads (`__fixtures__/claude/subagent-hook-payloads.json`) before changing any of this.

### Child hooks must not drive parent state

Claude child `PreToolUse` / `PostToolUse` / `PostToolUseFailure` hooks must not drive parent state. Child `PermissionRequest` and attention `Notification` hooks still reach needs-you and phone approvals, including the raw approval summary and deterministic reply ticket. Keep raw summary recording before the child transcript-association guard in both shells.

### Device checklist (not runnable in CI)

- (a) macOS and Windows canvas, interactive: cards at start, right labels, live activity, done at stop, nested card, resumed card re-opens.
- (b) SSH node: native tail over the ControlMaster at the derived path.
- (c) A session started before the upgrade (old hook snapshot; whether Claude reloads hooks mid-session is unmeasured): no double and no missing cards.
- (d) Eco with a background subagent paused on its own background shell: not hibernated, bulk restart skips it.
- (e) A managed-account node gets native cards (installer writes the account dir).
- (f) Windows: the derived path keeps the reported separator.
- (g) A pre-2.0.43 CLI tolerates the two new keys in settings.json (same class as `StopFailure`, which already shipped).
- (h) The hand-back turn (a background child reporting back wakes the parent for a turn, then the `<task-notification>` wakes it again) may chime "finished" twice; #708's quiet rule is per turn.

## Codex subagents

Codex (`spawn_agent` collaboration, issue #401) joined via its native `SubagentStart` / `SubagentStop` hooks, measured on codex-cli 0.146.0, keyed by `agent_id` (not `tool_use_id`: nothing correlates the spawn tool call with the Start it launches; `agent_id` is stable across the child's life, parallel and nested spawns included, and nested children fire through the same subscription so every card connects flat to the owning terminal node). Facts a refactor must not lose:

1. Every `agent_id`-tagged codex event carries the parent's `session_id` with the child's rollout as `transcript_path`. Both raw listeners skip the context-meter track for them (else the parent's meter re-points at the child), and `normalizeCodex` returns null for child tool events (else a child Bash event flips a finished parent back to RUNNING after an async spawn). Pinned by `hook-verified-parity.test.ts`.
2. The spawn task text is encrypted end-to-end (`tool_input.message` and the NEW_TASK payload are Fernet blobs), so there is no `taskLabel`; the readable `Task name:` header reaches the card via the activity stream instead.
3. The live tail is `subagentTail.trackFile` (the path is handed to us, no meta-dir matching) with the stateful, per-entry `createCodexSubagentFormatter` (`core/codex-subagent-format.ts`). A spawn child is a fork of the parent thread, so its rollout opens with a replay of the parent's context, suppressed until the `inter_agent_communication_metadata` / NEW_TASK gate. It is per entry because two concurrent subagents sharing one closure would gate each other.
4. Codex's `SubagentStop` is the real end (no async-launch-ack trap, no task-notification sniffing), carrying `last_assistant_message` as the card's result.

Remote (SSH) codex nodes get cards but no live activity yet: the child rollout is on the host, and claude's `remote-subagent-tail` has no codex counterpart (follow-up).

## Grok subagents

Grok joined via its own native `SubagentStart` / `SubagentStop`, measured on grok 1.0.13 by launching two `explore` children in parallel. Keyed by `subagentId`, occupying the same `toolUseId` slot the store already uses (claude by `agent_id` natively, else `tool_use_id`; codex by `agent_id`; grok has no tool call behind a subagent). Facts a refactor must not lose:

1. The start's `sessionId` is the parent's and the stop's is the child's own (equal to `subagentId`). Keying on it files start and stop under different cards and the started one never closes.
2. The child's transcript is derived from `subagentId` as `chat_history.jsonl` (`core/grok-subagent-format.ts`). The start's `transcriptPath` is the parent's, and even the stop names `updates.jsonl`, which parses to nothing.
3. A `session_end` bearing `subagentType` returns early in both raw listeners. Without that a child finishing tears down the parent's session state.
4. `description` arrives only on the start.

The four captured payloads live in `src/shared/agents/__fixtures__/grok/hook-payloads.json`, pinned by `normalize.grok.capture.test.ts`. Remote (SSH) grok nodes get cards from the hook but no live tail yet (the child dir is on the host; same gap as codex).

## Reload replay

`core/subagent-replay.ts` retains at most 512 running starts for the existing `WORKING_STALE_MS`, with original host timestamps, so a reloaded renderer can rebuild live cards. The shared mirror event path feeds it (including synthetic async ends); session boundaries and `clearNode` discard it. Both shells expose the same read, and desktop preload, browser and relay subscribe before requesting it. Only lifecycle events wait behind the snapshot (bounded to 512 events or 3 seconds); live permission alerts are not delayed or replayed. Not provided: disk restore, transcript backlog, completed-card replay, or authorization evidence. A host restart or a missed original start is still outside this recovery.

## /loop, /schedule and /cron node

Agents in `RECURRING_CAPABLE`. Detected from the tools the agent invokes (reliable; users often phrase it in natural language so the prompt rarely starts with the slash): `PreToolUse` for `Skill` (skill in loop/schedule/cron), `CronCreate` (cron, label = cron expr and prompt), or `ScheduleWakeup` (loop), plus a `UserPromptSubmit` `/loop|/schedule|/cron` prompt-prefix fallback, all surfaced as `recurring` normalized events.

- Sets `agentStatus.loop` (`{count, prompt, items, kind}`). For in-session `loop` each turn-done bumps the count and appends `lastMessage` (schedule and cron run in the background, so they are not counted).
- Lifetime by kind: `loop` dies with its session; `cron` / `schedule` outlive turns, sessions and app restarts (`loop` is persisted in the agentStatus localStorage) and are cleared by a `CronDelete` `recurring`-end event or the card's own × (which dismisses the card only; the recurring fact stays durable, see Eco in [agent-pause-hibernation.md](agent-pause-hibernation.md)).
- `clearForParent` (new turn) leaves the loop card's dragged position alone.
- Renders an ephemeral `LoopNode` labelled by kind, connected by an edge to the parent, plus a small header badge.

## Branch conversation

A node action (`IconBranch`, Claude-only via `BRANCH_CAPABLE`): sends `/branch` into the existing terminal via `pty.sendText` (tmux `send-keys`) and opens a new Claude node that resumes the parked original with `claude --settings … -r <ORIGINAL_ID>`. The original id is the session id already known from hooks; `lib/claudeBranch.ts` is the fallback that parses `pty.capture` output when the id is not known. The source node stays on the new branch.
