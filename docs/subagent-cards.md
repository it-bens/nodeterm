# Subagent cards, loop cards and Branch

How the canvas draws subagents (ephemeral `SubagentNode` cards), recurring `/loop` / `/schedule` / `/cron` work (`LoopNode`), and the Branch conversation action. Covers the normalized subagent events from Claude's native hooks and the older tool-call reconstruction, the codex and grok variants, the live tails, the reload replay, and the removal rules that Eco's hibernation guard depends on. The hook transport and normalizers are described in [hook-model.md](hook-model.md).

## Subagent visualization

Agents in `SUBAGENT_CAPABLE` emit `subagent-start` / `subagent-end` normalized events, which drive a transient `state/agentNodes.ts` store. For Claude they come from Claude's own `SubagentStart` / `SubagentStop` hooks whenever a session sends them (below). The older reconstruction is kept as the fallback and as the source of the task label: `PreToolUse` / `PostToolUse` on tool `Agent` / `Task` correlated by `tool_use_id`, whose PostToolUse on an async launch is only an ack (`status:'async_launched'`), with the real end sniffed from the `<task-notification>` queued into the parent transcript (context tails produce a synthetic `subagent-end` in both shells). Neither the notification's `UserPromptSubmit` nor the `[Subagent hand-back]` one is a `newTurn`, so neither clears the fan-out.

Canvas renders each subagent as an ephemeral `SubagentNode` (display-only card: type, task, working/done) connected by an edge to its parent agent node. These nodes and edges live outside the React Flow `nodes` state (merged only at the `<ReactFlow>` prop), so they are never persisted (`flowToNodeStates`) nor in undo/dirty. Subagents share the parent's process: no PTY. Each card shows duration, tokens and tool-uses, and expands (click) to a live transcript: `core/subagent-tail.ts` tails the subagent's own transcript file (`<…>/<sessionId>/subagents/agent-<id>.jsonl`; for a native card at the path derived from the parent's transcript and the `agent_id`, `claudeSubagentTranscriptPath`; for a tool-path card matched by `tool_use_id` via the sibling `.meta.json`), read-only, formats each line (assistant text, tool calls, results), and streams chunks over `agent:subagent-activity` into the store. That text stream is what every card but a native Claude one shows when expanded; a native Claude card shows its conversation instead (see [a native Claude card reads as a conversation](#a-native-claude-card-reads-as-a-conversation)).

### Two different clears, and the difference is load-bearing (issue #547)

- The removal paths (node delete, project delete, the cross-project close, the orphan-session kill, `SessionEnd`) call `clearForParent`, which drops everything: a node that is gone has no work left to represent.
- A new turn calls `clearFinishedForParent`, which drops only `state === 'done'` cards that are not expanded (an open card is pinned, see [an open card is pinned](#an-open-card-is-pinned)). "The previous fan-out is stale by definition" is true of a finished card and false of a working one. Claude launches subagents async, so "waiting for N background agents to finish" is exactly the state in which the next prompt gets typed, and nothing rehydrates `byId` afterwards (`start()` fires only from a live launch event; a running subagent emits no second one). The card was gone for the rest of the run while the agent kept working. The expensive half is not the missing card: Eco's hibernation guard derives `liveSubagents` from this same store, so the wipe let a parent with live background agents read as idle and get its CLI `/exit`ed.
- Keeping an unfinished card owes a decay: `useAgentNodes.sweepStaleWorking`, on the same 60 s tick and the same `WORKING_STALE_MS` as `agentStatus`'s (imported, never re-chosen: `shared/agents/stale.ts` exists because three surfaces each invented their own timeout). Otherwise a subagent whose end never arrives pins its card, and its parent, forever. It marks the card done rather than deleting it, so a late `finish()` is the no-op it already was and the next turn boundary takes it.
- **A third removal path is opt-in:** `settings.autoHideFinishedSubagentCards` (default off; off reproduces the two paths above exactly) mirrors into the store as `autoHideFinished`, and with it on a card is dropped the moment its subagent reports done, with no turn boundary. Only a done card is ever dropped, and never an expanded one (`finish()` only turns it done), so #547's rule and Eco's `liveSubagents` are untouched. The decay is deliberately not one of the paths it gates: `sweepStaleWorking` fires precisely because the end never arrived, which is the opposite of what the setting promises, and it is the one case where a visible card carries the most information (drop it and a fan-out whose subagents died silently leaves nothing on the canvas saying one was ever launched). It costs Eco nothing either way, since an absent card and a done card are the same answer to `liveSubagents`. Renderer only: Desktop and Server Edition identical, Mobile N/A.

### An open card is pinned

- Expanding a card is reading it, so the turn boundary (`clearFinishedForParent`) and `finish()`'s `autoHideFinished` drop both skip an expanded card, which only turns done.
- Collapsing a finished card un-pins it: it goes at the next boundary, or at once under `autoHideFinished`. `toggleExpanded` drops it in that case, including a card the stale decay marked done, because the store cannot tell the two kinds of done apart.
- Tidy (`tidyFanout`) drops every `expanded` flag on purpose and follows the same rule.
- `clearForParent` (the parent is gone or its session ended) still takes a pinned card, and `sweepStaleWorking` still only marks it done.
- Eco is unaffected: a pinned card is `done`, and `liveSubagents` is "any card not done" (`lib/hibernationCandidates.ts`). The bulk idle restart never reads the card store; it reads agent state and `backgroundTaskAt`. See [agent-pause-hibernation.md](agent-pause-hibernation.md#eco-hibernation).
- The pin lives in memory like the cards: a renderer reload or an app restart loses an open card (a running card comes back through the [reload replay](#reload-replay), closed). There is no cap on pinned cards, because only an expanded card mounts a panel.

### A native Claude card reads as a conversation

An expanded card that is `chatReadable` shows the conversation instead of the text stream. `chatReadable` needs all of: its `agent` is the claude harness (by `capabilityAgentId`); it is `native`, meaning it was drawn from `SubagentStart` with `subagentSignal === 'native'`, so its key is the child's `agent_id`, a safe file-name token (`isClaudeAgentId`); and it recorded the parent's `sessionId`. The card body is then the [⌘M `ChatPanel`](chat-view.md), read-only, pointed at the subagent's own file: `subagent={{agentId: cardId, working}}`, with `nodeId`, `sessionId` and `accountId` the parent's and `quoteTargetNodeId` the parent. A selection in the thread offers "Add to chat" to the parent ([quote to chat](quote-to-chat.md)); the subagent has no message box. Every other card (Codex, Grok, a tool-path card, no recorded session, a closed card) keeps the text stream, unchanged. Rules a refactor must keep:

- **The parent's identity is recorded at start.** `SubagentViz.sessionId`, `accountId`, `agent` and `native` are set by `applyAgentEvent` from the event and the parent's fresh status entry. The first recorded value wins and `native` is sticky once true, so a later resume, restart or account switch of the parent never repoints an open card, and a replay without the signal cannot demote one.
- **A parent restart or account switch ends the card.** An in-place restart or an account switch quits the parent's CLI, which fires `SessionEnd`, so `clearForParent` removes the card, pinned or not. An opened card does not survive a parent restart.
- With no recorded account the card reads under the parent node's own rule (`effectiveAccountId(data.accountId, observed, claudeAccounts)`, the same one TerminalNode's `accountForReads` uses). `core/subagent-replay.ts` carries `subagentSignal`, so a running card replayed after a reload stays native; a finished one is not replayed.
- **The size is a layout default, never stored.** `subagentCardSize` lays an open chat card out at 480 x 560, an open text card at 480 x 340 and a closed one at 230 x 96; a size the person set wins. A stored size is also the closed size, so storing 480 x 560 on open would leave a big empty frame around a closed header.
- **Live reads come from the card, never the parent.** `useSubagentReloadNonce` bumps ChatPanel's `reloadNonce` when the card's `lastActivityAt` advances (subscribed outside render; at most once per `CHAT_LIVE_RELOAD_MIN_MS`, with a trailing bump, through the panel's own throttle, single flight and hidden-document hold). It bumps again on the working to done edge at 0 ms and at each `TURN_END_RELOAD_DELAYS_MS` delay, because a stop can precede the last record. The panel skips the parent's `onHookEvent` and turn-end reloads (its `nodeId` is the parent), its `liveTurn` follows `subagent.working`, and as a read-only panel it does not subscribe to the parent's draft. The transcript identity gains the subagent id, so the thread never merges with the parent's.
- The file's first record renders as a labelled Task block (`ChatMessage.task`), not as the person's bubble. A finished card puts its report (result, then the stats line) under the thread. The stats line is not `nodrag`, because it is the one strip left to drag the card by.
- Read-only views never count as an open chat view: not for a quote's `chatOpen`, and not for dictation (`LIVE_CHAT_VIEW`, see [focus, pickers and dictation routing](chat-view.md#focus-pickers-and-dictation-routing)).

### Reading a subagent's file: `chat.readTranscript`'s 7th argument

`chat.readTranscript` takes an optional 7th argument `subagent: {agentId}` (preload, ws-bridge `buildTranscriptApi`, the core handler). The relay stub still refuses it.

- **Validation.** Core (`readChatTranscript`) validates it first with `normalizeSubagentArg`: absent or `null` is the ordinary read; anything else must carry an `isClaudeAgentId` token, or the call throws `Invalid subagent`. It is served only paged, and only when the parent's agent is stated and resolves to the claude harness (`capabilityAgentId`; an omitted agent is not assumed to be claude here). Anything else is not found.
- **The parent** is resolved strictly by session id (`resolveTranscript` with no cwd, so never the cwd's newest session) and must be the session's own file (`isSessionTranscript`: the last segment is `<sessionId>.jsonl`, separator-agnostic). A stale hook-fed path naming another session would otherwise put that session's `subagents/` directory in reach.
- **The child** is derived with `claudeSubagentTranscriptPath`, then jailed.
  - Locally (`core/subagent-transcript-path.ts`, `subagentTranscriptFor`): the child directory's realpath must equal the parent directory's realpath plus `<session>/subagents`, and `lstat(child).isFile()` must hold (no symlinked file or directory; a `~/.claude` that is itself a symlink still works).
  - Remotely (`main/remote-transcript-page.ts`): the same derivation from the host's parent ref, jailed by the parent's own host jail (`isSafePath`, which is `isSafeRemoteTranscriptPath` against that master's remote home). The dep is required, so a shell that forgot it reads no subagent. The read uses the ranged SSH reader. A remote failure is unreadable, never this machine's disk, and a failed subagent read keeps the located parent ref (the child may simply not exist yet).
- **Limits, stated.** The remote jail is lexical: a symlink at the derived host path would be followed, and planting one needs the host user's own `~/.claude`. Locally the `lstat` to `open` window is a same-user race (`readChatWindow` opens without `O_NOFOLLOW`).

### The subagent-file parser

`ChatParseOptions.subagentFile` is honoured only with `rich`, and core sets it in one place: the `parse` of a read that carries `subagent`. The phone's `chat.page` never sets it, so the wire format is unchanged.

Measured on Claude Code 2.1.326: three background subagent files from a controlled run, one fork's first line, and the CLI bundle's own strings and identity rule. The fixture `src/shared/chat-fixtures/claude/subagent.jsonl` is hand-built from those shapes with every value replaced (see "Subagent files" in `src/shared/chat-fixtures/README.md`).

- Every record carries `agentId`, and every conversation record (`user`, `assistant`, `attachment`) also carries `isSidechain: true`. The `api-request*` records and a fork's `fork-context-ref` do not.
- A background subagent's first line is its task: a `user` record with string content and `parentUuid: null`. A fork's first line is `fork-context-ref`, then the parent's replayed `Agent` call and a `[tool_result, text]` user record.
- A person's message to a running subagent is an `attachment` `{type:'queued_command', prompt, source_uuid, origin:{kind:'human'}, isMeta:true}`. To a finished (revived) one it is a `user` record with `isMeta:true` and `origin.kind:'human'`, wrapped in the CLI's `SUBAGENT_MESSAGE_HEAD` / `_TAIL`. Neither shape reaches the parent's transcript.

Readings the parser applies:

- The file's first non-meta `user` record is the task (`task: true`) when it is a sidechain record, and only when the read holds byte 0, so an unpaged caller must hand the whole file.
- Both person shapes become the person's bubble, with reminder markup and the wrapper removed (head and tail both exact, the CLI's own unwrap; the head alone as a fallback).
- One bubble per message by the CLI's identity rule (`uuid`, an attachment's `source_uuid`, a user record's `sentAsUuid`). A record's ids are recorded only when its bubble is drawn, so a form that unwraps to nothing cannot hide the other form.
- These readings apply only to a subagent read: in a main transcript an `isMeta` queued attachment is a peer message.
- A fork's replayed parent `Agent` call still shows as a tool chip before the task (left as is in v1).

Not measured: both forms of one message in one file (the dedupe is tested on inline records only), array-content person messages (mirrored from the CLI's rendering), and a duplicate split across two pages (drawn twice).

### Listed under the parent

- **Where.** The sessions sidebar draws one indented row per card under the agent's session row, in both modes (`components/SubagentRows.tsx`: type and task on one line, then a working dot or a done mark). The kanban session card shows "N subagents · M working" (`kanban/SubagentChip.tsx`), and the card modal shows a compact list.
- **One subscription.** All read the same `useAgentNodes` store through `subagentRowsSig` (`lib/subagentRows.ts`), never `byId`. The signature is a primitive built from the raw fields in store order (id, state, start time, and the length-prefixed type and task; no regex, no sort). It carries the label because the lifecycle relabels a running card under its own id.
- **The working mark is a static dot, never a spinner.** One infinite animation per row keeps a focused window's frame loop open (#1454; see [idle energy](idle-energy.md)), and `SubagentRows.test.tsx` fails on an `animation` in any rule for these rows.
- **Opening a card.** A row click (or a list click in the card modal, which first closes the modal and calls `showCanvas`; the conversation is never drawn inside the board) dispatches `nodeterm:open-subagent`. Canvas's one listener calls `travelToNode` for the parent, which answers whether it travelled (`false` when blocked or when a reopen was declined). Only then does it open (never toggle shut) and select the card, at once rather than when the travel lands, since opening pins it. The camera frames the parent; on a parent that fills the screen the card can sit just out of view.
- **Hidden fan-out.** A parent whose eye hides its fan-out (`hideFanout`) still gets the card opened, plus an info strip naming the eye (`SUBAGENT_FANOUT_HIDDEN_TEXT`). `parentHidesFanout` is read before the travel, from the live canvas, else from the owning project's stored copy. The flag is persisted node data and is never written from there.
- **Never for a relay or team project.** Sidebar rows and the modal list ask `isRemoteProjectId`; the board chip gets `SessionCard.relayProject` from `project.remote`. The card store is this machine's and keyed by node id, a relay tab draws no cards at all (`wireRelayAgentStatus` feeds `applyAgentEvent` with `nodes = null`), and a team tab shares its node ids with the handed-off SSH project.
- **Surfaces.** Desktop and Server Edition are identical (renderer plus the core read; the Server Edition reads locally, since it runs on its host). SSH projects read the card's file on the host. Mobile: N/A (no subagent cards). `src/shared/chat-fixtures/claude/` is a fixture directory the iOS repo's copy will see, with nothing to implement there, because the phone never sets `subagentFile`.

### Device checklist for the conversation card and rows (jsdom only so far)

- (a) The 480 x 560 chat card in dark, light and Liquid Glass.
- (b) A wheel over the thread scrolls it, not the canvas.
- (c) "Add to chat" from a card's thread into the parent, both legs.
- (d) A working subagent's thread grows in the open card.
- (e) An SSH node's card reads on the host.
- (f) The Server Edition forwards the 7th argument (pinned at source level only).
- (g) One person's message written in both forms draws one bubble.
- (h) Sidebar row spacing in dark, light and Liquid Glass, the chip in a crowded card title row, the modal list's place.
- (i) Dragging a finished chat card by its stats line.
- (j) On the Omni board, a list click for a background lane switches project and frames the parent (the synchronous modal to canvas hand-off is unverified).
- (k) A row click on a parent with hidden cards shows the info strip, and the card is open once the cards are shown.
- (l) After a reload an open card is gone, and a running one comes back native through the replay.

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

## Messaging a subagent (measured, not built)

Measured on Claude Code 2.1.326 (haiku, a private tmux socket, a scratch cwd, `NODETERM_*` unset, 2026-11-17): a person can message a running background subagent from the TUI.

- **Keys.** With the prompt empty: Down (focus enters the `⏺ main` / `◯ <type>` list the footer's `↓ to manage` announces), Down again (the subagent row: "enter to view · x to stop"), Enter (its view, with the composer placeholder `Message @<type>…`, the agent type and not its description), type, Enter. Back: Down (main) then Enter; Up from the top row only leaves the list.
- **Which subagents.** A finished subagent whose row is still listed is revived by a message (`revivedAt` in its `meta.json`); the row is gone once the main turn ends. Plain Agent-tool subagents accept messages, not only teammates. A foreground one could not be made: an Agent call without `run_in_background` was still backgrounded (meta `requestShape:"background"`).
- **Where the message goes.** It is recorded only in the subagent's file (the two shapes the card's parser reads, above) and fires no hook, not even `UserPromptSubmit`. The main agent never learns of it: it sees only the handback ("Message from @<type>") and a task notification, each costing it a turn.
- **Hazards for anyone who drives this with keys.** `x` in the list stops the selected agent. A running subagent's view says `esc to stop` (Esc likely stops it, untested). Left arrow on an empty main prompt opens a new-session composer (text plus Enter starts a session). Rows are positional and not unique (two can read `general-purpose`), and their status text goes stale.
- **Consequence.** A future "Message @agent…" box in the card would need a screen read before every key and works only while the row is listed. The read side, the person's message drawn in the card's thread, already exists.

## Reload replay

`core/subagent-replay.ts` retains at most 512 running starts for the existing `WORKING_STALE_MS`, with original host timestamps, so a reloaded renderer can rebuild live cards. Each replayed start keeps its `subagentSignal`, so a running native card stays native (and conversation-readable) after the reload. The shared mirror event path feeds it (including synthetic async ends); session boundaries and `clearNode` discard it. Both shells expose the same read, and desktop preload, browser and relay subscribe before requesting it. Only lifecycle events wait behind the snapshot (bounded to 512 events or 3 seconds); live permission alerts are not delayed or replayed. Not provided: disk restore, transcript backlog, completed-card replay, or authorization evidence. A host restart or a missed original start is still outside this recovery.

## /loop, /schedule and /cron node

Agents in `RECURRING_CAPABLE`. Detected from the tools the agent invokes (reliable; users often phrase it in natural language so the prompt rarely starts with the slash): `PreToolUse` for `Skill` (skill in loop/schedule/cron), `CronCreate` (cron, label = cron expr and prompt), or `ScheduleWakeup` (loop), plus a `UserPromptSubmit` `/loop|/schedule|/cron` prompt-prefix fallback, all surfaced as `recurring` normalized events.

- Sets `agentStatus.loop` (`{count, prompt, items, kind}`). For in-session `loop` each turn-done bumps the count and appends `lastMessage` (schedule and cron run in the background, so they are not counted).
- Lifetime by kind: `loop` dies with its session; `cron` / `schedule` outlive turns, sessions and app restarts (`loop` is persisted in the agentStatus localStorage) and are cleared by a `CronDelete` `recurring`-end event or the card's own × (which dismisses the card only; the recurring fact stays durable, see Eco in [agent-pause-hibernation.md](agent-pause-hibernation.md)).
- `clearForParent` (new turn) leaves the loop card's dragged position alone.
- Renders an ephemeral `LoopNode` labelled by kind, connected by an edge to the parent, plus a small header badge.

## Branch conversation

A node action (`IconBranch`, Claude-only via `BRANCH_CAPABLE`): sends `/branch` into the existing terminal via `pty.sendText` (tmux `send-keys`) and opens a new Claude node that resumes the parked original with `claude --settings … -r <ORIGINAL_ID>`. The original id is the session id already known from hooks; `lib/claudeBranch.ts` is the fallback that parses `pty.capture` output when the id is not known. The source node stays on the new branch.
