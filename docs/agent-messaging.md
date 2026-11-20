# Agent messaging and the agent status store

How agents (and the person) send text into other agents' panes, how a send is queued or refused, how a board comment that @mentions a session becomes a message, and the shape of the per-node agent status store the messaging layer reads. Main code paths: `core/agents/agent-messaging.ts`, `renderer/lib/messageScopeSync.ts`, `@shared/board-comment`, `renderer/state/agentStatus.ts`.

See also: [adding an agent](adding-an-agent.md), [hook model and identity](node-identity.md), [durable orchestration state](durable-orchestration-state.md).

## Agent status store

The app is a pluggable multi-agent system: Claude Code is one builtin of several. Extra terminal-node behavior is driven per agent by a registry plus capability lists (see [adding an agent](adding-an-agent.md)), a shared 4-state model, and a **transient** zustand store `state/agentStatus.ts`.

- Per node id the store holds `{state, agentId, unread, session, sessionId, loop, hibernated}`. The live `state` is **not** persisted. Only `unread`/`session`/`sessionId`/`agentId`/`loop`/`hibernated` go to localStorage under `nodeterm.agentStatus` (migrated once from the legacy `nodeterm.claudeStatus` key).
- `agentId` is durable because a hand-launched `claude` in a plain terminal is known nowhere else, and its context links must keep classifying across restarts.
- That is the DEFAULT instance, the local core's. A relay or team tab's session gets its own **keyless** store (`agentStatusForApi`): it persists nothing, never acks a read (no `ackDone` callback), and is fed by `applyAgentEvent` (`state/applyAgentEvent.ts`), the same reducer Canvas's listener uses. See "A relay session owns its agent status" in [hosted-team-relay](hosted-team-relay.md).

## Message scope publication

Desktop `send`/`reply`/`notify` wait for pending active-canvas edits to be saved when either endpoint is on that canvas (`renderer/lib/messageScopeSync.ts`).

- `list` and context links can already see a newly opened node while main's `persistedCanvases()` cannot; the scope resolver reports that absent target as `cross-project`. The barrier closes that gap.
- The publication barrier never travels, never overrides an external-edit conflict, and does not authorize anything: main still checks unique project membership, runtime ownership, consent, verified status and the native pane.
- An unrelated active canvas is not saved for a background message.
- Server control already writes its nodes through the authoritative store; the renderer barrier is a desktop concern. Mobile is not an agent-message sender.

## A `send` to a node that has not started yet is queued, not refused

Outcome `targetNotStarted` (`agent-messaging.ts`). A node opened into a project that is not on screen without `--run-now` exists only as a held launch until the project is viewed, so no spawn has recorded its pane owner.

- When the owner is unproven AND no session exists AND this machine holds a launch for it (`WorkspaceStore.heldLaunch`, the machine-local `localExec` overlay), the outcome is `targetNotStarted` and the deliver-on-idle queue holds it. The flush re-runs every gate against the pane the spawn will have proven. Both shells wire `heldLaunch`.
- A LIVE pane with no proven owner stays `unproven-target-owner`. That refusal is the security property.
- Such a message waits up to 24 hours (`NOT_STARTED_TTL_MS` = `QUEUE_PERSIST_TTL_MAX`): the start waits for a person to open the project, and the ordinary 5-minute TTL lost the message in the field (queued 19:12, expired 19:17, project opened 19:27).
- It is not carried across an app restart (no session was recorded to bind it to), and a node deleted before it starts keeps it until the TTL.
- A `targetStatusStale` target (a station started a moment ago by `--run-now` or `run` that has not posted its first hook) is queued too, with the ordinary TTL: a retry cannot help before that hook, and its first verified `done` flushes the queue.

## A board comment that @mentions a session is a message from a person

`@shared/board-comment`, `deliverBoardCommentFromUi` in `core/agents/agent-messaging.ts`.

**Token and delivery.** The comment composer's @ picker inserts an id-based token `@[label](node:<id>)` (the id is the authority, the label only a fallback name). On send, each mentioned session gets its own delivery through the SAME `runDelivery` an agent `send` takes: scope, runtime pane ownership, the per-project `agentMessaging` switch, flow control, the pane probes, the nonce envelope, the receipt, the deliver-on-idle queue with its TTL.

**What differs is only who it is from:**

- The scope is "the target is on the comment's board" (`resolveBoardCommentScope`).
- The PAIR window belongs to the board (`board:<projectId>`: one comment per session per 10 s, whichever comment). The FAN-OUT budget belongs to the comment itself (`reserveFlow`'s `fanOutKey`, `board:<projectId>:<commentId>`), because a person's turn is one comment and an earlier comment's in-flight holds or queued flush must never spend a newer one's (both scenarios are tests).
- A comment mentions at most `BOARD_COMMENT_MENTION_MAX` = `FANOUT_PER_TURN` sessions and is refused whole above it.
- A pair-limited board comment is QUEUED rather than refused (`BOARD_QUEUE_ON`: an agent retries, a person could only post again; the flush re-runs the limiter) AND re-offered on a timer when the window ends (`DeliveryQueue.retryAfter`, armed at enqueue and at every `rateLimited` re-queue, one pending nudge per target). The queue otherwise flushes only on the target's `done`, and a window ending emits nothing, so a session already idle, or whose `done` landed inside the window, left the comment to expire at the TTL.
- The envelope reads `from: board comment by <author>` with no node id and `reply-to: none (…)` (`BOARD_COMMENT_REPLY_TO`; both agent-facing bodies render it from the constants). An agent node TITLED like that is labelled `node titled "…"` in its own `from:` line, so it cannot pass as a person.
- The body is the comment with each token turned into the `@<name>` its author saw (`mentionNameForAgent`: the token's label reduced to letters, digits, spaces and `. _ - #`, capped at 40, else the node id; a node title is whatever the project file says, and it lands in another agent's prompt), then `sanitizeChatText` (every C0/C1 control but `\n`/`\t`, the one shared rule) and a cap.
- A comment's parallel mention deliveries share ONE `syncMessageScope` save (`coalesce`).

**Only the local user, typing in this app, can trigger it.** A comment in the board log is text, never a trigger. The log is a shared file: a git pull, another instance, a relay peer or a team-presence guest can put a token in it, so nothing that loads, reloads or renders it may reach a delivery.

- The one call site is `BoardLogPanel`'s send (`board-comment-trigger.guard.test.ts` fails on a second call site).
- The IPC (`agent:board-comment-deliver`) is a raw, main-window-only `ipcMain` handler that no peer can dispatch into (never on the peer-dispatchable platform table) and is also `HOST_ONLY`.
- The renderer refuses a relay-bound project and a browser tab (`canDeliverBoardComments`).
- Around the IPC, Canvas takes the same two steps as `send`: the target's `guardConcurrentRestart` lock and `syncMessageScope`.

**No silent success.** Every outcome is on the comment row: `sending…`, then the reply's typed outcome (with the `notPermitted` reason) from this app run, else the latest `agent-message` trace line in the log whose `from` is `board-comment:<commentId>`. The trace carries `reason` too, and the queue hands its trace leg the queued request, so a board comment's `queued`/`expired` lines land on ITS board whoever owns the pane by then.

- Log lines are trusted ONLY for comments this machine sent (`nodeterm.boardCommentsSent`, localStorage, bounded). The log is shared, so a teammate's comment arrives with THEIR machine's trace lines and a forged line is one append away: such a comment shows no status, and its lines stay ordinary feed rows.
- For our own: a line dated in the future is ignored; a `queued` older than `BOARD_COMMENT_QUEUE_STALE_MS` (the queue TTL + 1 min; a crash still never writes its end, a clean restart does, since the durable queue expires a restored board comment at boot) says no outcome was recorded; a mention with no record at all says so too.
- A trace line is hidden as a row only on the card whose comment row shows it; the mentioned session's own card keeps "routed a board comment here: …".
- Text tables are read with `Object.hasOwn` (values come from the shared file).
- The card modal's capture-phase Escape defers to the composer while its @ picker is open.

**Surfaces.** Desktop: full. Server Edition and relay tabs: display-only by design (the bridge answers `notPermitted: unsupported-edition`; the picker is not offered): a browser or a relay guest typing into this machine's panes is exactly the cross-user injection this refuses. Mobile: N/A (the phone posts no board comments).

## Writing refusals and side effects

- **An error must not name a remedy nobody measured, and the remedy needs its own test.** The `unproven-target-owner` refusal told its caller "Re-open the target node so its owner is recorded, then try again", and re-opening is precisely the attach that records nothing (ownership is recorded only on a genuine fresh spawn), so a caller that obeyed got the identical refusal forever. It shipped because no test read the sentence. When you write a refusal, pin its claim against the MECHANISM it describes: assert the remedy's precondition by calling the function that decides it, so the copy goes red when the behaviour moves. Remember who reads it: telling a language model to do something only a human can do is not advice.
- **A side effect you could not deliver is not a side effect you performed.** The `ok:false` rule is not only for reads. `catch {}` around a remote kill folded "tmux says there is no such session" (an ANSWER) into "the ControlMaster is down" (a NON-answer, session still running). Classify the failure, and when the work genuinely cannot be done now, either refuse the action with a reason or write the debt down and settle it later (`core/pending-remote-kills.ts`). Silently dropping it is the one option that is never available.
