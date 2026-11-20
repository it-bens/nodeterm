# Notifications, status ordering and session names

How a finished or blocked agent turn reaches the user (unread marks, OS notifications, Dock/taskbar badges, sounds), how the sessions list orders and ages rows across restarts (`lastSeen`), how the agent's session name and the node title stay in sync, and the rules for UI that reads agent status. The hook events that feed this are described in [hook-model.md](hook-model.md); the shared 4-state model lives in the agent registry docs.

## Unread and OS notification

On a busy to idle edge while the window is unfocused (`document.hasFocus()`), the node is marked unread (header dot, minimap stroke, project-tab dot). If notifications are enabled, `window.nodeTerminal.notify()` goes to main `app:notify`, shown only when `mainWin.isFocused()` is false. Clicking it focuses the window and sends `app:focus-node`, which calls `Canvas.focusNodeById` (selects and centers, switching projects via `pendingFocusRef` if needed). A one-time consent prompt gates notifications; the toggle is in Settings (`notifyOnClaudeDone`).

Selecting, focusing, dwelling into, or opening a session card clears `unread` and ACKs the finish across phone and notch surfaces (read-on-view). This never changes the workflow bucket: read state is independent from agent state.

### Badges

- The unread count is the macOS Dock badge. It is counted over nodes some project holds (closed projects included), never over the raw status table, which outlives nodes deleted by paths that skip `remove`. An orphaned entry is swept at load once its 90-day `lastSeen` clock ages out, and the sweep is skipped whenever the load is incomplete (`lib/unreadBadge.ts`).
- On Windows: a taskbar overlay icon (pure `planBadge` plus BGRA rasterizer `rasterizeBadge`, "9+" above nine, `main/taskbar-badge.ts`) and a `flashFrame` when the count rises while unfocused (cleared on focus). Linux shows nothing.
- Windows toasts need `app.setAppUserModelId` set before ready to the package.json `build.appId` the NSIS shortcut carries (`main/app-identity.ts`, pinned by test; `execPath` in dev). Without it toasts are dropped or mis-attributed silently (Windows never emits Notification `failed`).

## Sound alerts and custom sounds

The `done` / `needsYou` alert (`SfxKind`) is a synthesized WebAudio chime (`renderer/lib/sfx.ts`, fired from Canvas's `alert` closure, gated by `soundEffects`, 5 s per-node cooldown). Settings → Notifications lets the user replace either with their own file.

- The picked file's bytes (never its path) go to `files.saveAlertSound`, a core handler in `registerFsHandlers` (both shells), which validates kind, extension allow-list, 5 MB cap and magic bytes, and writes one format-independent name `<userData>/sounds/<kind>.sound` (`core/alert-sounds.ts`). A name per format needed a delete-the-others step, and two tabs saving different formats at once deleted each other's file, so do not reintroduce per-format names. Reads and Reset take only the kind (no path from the renderer, symlinks refused).
- Settings keep only `customAlertSounds[kind] = {name, stamp}`.
- Playback decodes the bytes with `decodeAudioData` (no `<audio>`, so CSP `media-src` is untouched on both surfaces). **Any failure (missing file, refused read, decode/playback error) falls back to the chime without throwing** (`lib/customSfx.ts`). Read failures and a missing or closed audio context are retried on the next alert; a decode failure is cached for that `stamp`, so a new pick is tried afresh.
- Every `playSfx` caller must pass `customAlertSounds` (source-pinned in `customSfx.wiring.test.ts`).
- Server Edition: full; the browser's `<input type=file>` bytes are stored in the server's data dir. Mobile: N/A (own notification sounds).

## Status-grouped sessions

Three always-visible sections: **Waiting for your response** maps internal `done`, `waiting` and `blocked` together (a completed turn, question or approval all need the user); **Running** maps `working`; **Unknown** means no live hook state is available. There is no Done bucket: a normal `done` hook means the turn ended and the agent is waiting for another user prompt. Within each section rows sort newest-first by `lastEventAt`, the transition clock (same-state hook freshness is `stateAt`), and show its short relative age. Missing clocks stay last with no made-up timestamp. A click may clear the glow but cannot move the row.

### The clock survives a restart as "last seen", never as a state

`agentStatus.lastSeen` is `{at, state}`. `lastEventAt`, `stateAt` and `state` are transient, so before this every row after a restart sorted as "no clock" and lost its age. `lastSeen` is the time of the last hook event (a same-state one included) and the state it asserted, persisted beside the agentStatus record under its own small key (`nodeterm.agentStatus.lastSeen`). It was chosen over the core mirror because the sidebar already reads this store, the mirror expires state after 6 h and identity later, and reading it would need a new IPC leg for a display fact. Rules a refactor must not undo:

- **Restored as a clock only.** Load fills `lastSeen` and nothing else: `state` stays unknown (the hook server was down with the app, so a turn may have started or ended in between), and `lastEventAt` stays unset. Load marks the clock `restored` (transient, never written). The row reads `lastEventAt ?? lastSeen.at` and its `statusClock` is `transition`, `seen` (a hook event this run but no transition yet) or `restored`. Only `restored` says "before nodeterm restarted" (`seen 3h ago`, tooltip naming the state it was last seen in).
- **The first event after a restart is usually a same-state one** (a cold-restore `--resume` fires SessionStart = `state: undefined` on an entry whose state is already unknown), so the store's in-place fast path must not take it. It replaces the entry (the row loses `restored` and re-sorts at once) without stamping `lastEventAt` (an unknown state is not an idle clock).
- **Eco never reads it** (so `idleKnown` and `planHibernation` are unchanged). Even a proven-idle prompt after boot would not be enough: the background-task stamp and the subagent cards Eco also needs are transient and cannot be rebuilt after a restart, and `/exit` kills both silently. A session becomes a candidate again from its next live `done`.
- **Bulk pause is the one reader that acts on it** (`idleForPause`, `lib/bulkPause.ts`): an unknown-state session is pausable only when `lastSeen.state === 'done'`. The difference from Eco is consent and proof, not the clock: a person confirmed the pause, the runner still proves the agent owns its pane before typing, and without it every session after a restart would be unpausable exactly when memory is tightest. Residual: a session last seen idle that started a turn while the app was closed (cron, loop, task notification) can be interrupted. Nothing automatic may read it.
- **Its own key, so the main table's write cadence is unchanged.** State events still write nothing to `nodeterm.agentStatus`. That matters twice: the main table carries `loop.items` (up to 100 x 4000 chars per loop node: 428 KB and about 2.1 ms per stringify with one full loop, measured), and on the Server Edition every tab rewrites its whole in-memory table, so a periodic rewrite would let tab B undo tab A's `clearUnread` / `hibernated` within seconds. A first version stored the clock inside that table on a 2 s throttle and rewrote it every 2 s while any agent worked. The clock key is saved on a real trailing debounce (5 s quiet, `LAST_SEEN_SAVE_MAX_WAIT_MS` 30 s at most while never quiet) plus `pagehide`: 2 Hz hook events for 10 s produce one write. Measured: about 59 bytes per clock, 1000 clocks = 59 KB and 0.7 ms per `JSON.stringify`. Two Server Edition tabs still race on the clock key (last writer wins), which costs only display.
- **Bounded.** At most `LAST_SEEN_MAX` (1000) newest clocks are written. A clock older than 90 days or more than 5 min in the future is refused on load (hand-editable input; a future stamp would pin a row to the top). An unknown `state` keeps the time and drops the state. An unreadable clock key costs the clocks, never the table.
- **It cannot create a row**: rows come from canvas nodes, never from the status table. `remove(id)` writes the clock key at once, a pending debounced save included. A deletion path that bypasses `remove` (for example `reloadActiveProject` dropping nodes) leaves a clock that simply ages out. Tests: `state/agentStatus.lastSeen.test.ts`.
- Surfaces: Desktop full; Server Edition per browser profile (localStorage, like `unread`); relay tabs keep a keyless store and persist nothing; kanban has no clock-ordered view; Mobile N/A (its own state).

## Session name and node title

**Two lists, because the two directions are separate facts.** `TITLE_READ_CAPABLE` (`canReadTitle`: claude, codex, grok, gemini) is the read leg, `RENAME_CAPABLE` (`canRename`: claude, grok) the write leg, and **read is a superset of write**, an invariant pinned in `config.capabilities.test.ts`. Gemini and codex are the reason: they name their own sessions (codex via `readCodexSessionName`) but have no rename command (gemini's `/chat save <tag>` is a checkpoint, not a title), so one list for both legs would light the rename UI on a node where the write silently does nothing.

The write is the same literal `/rename <name>` for claude and grok. The read legs are per-agent and none may ever search another's tree, so the routing lives in one place, `core/agent-session-name.ts` (`readAgentSessionName(sessionId, accountId?, agentId?, deps?)`, trailing and optional so every pre-grok caller is unchanged), serving the desktop IPC handler and both shells' session-name sweeps.
- Grok's read leg is `core/grok-session.ts` over `summary.json` in the session dir a hook told us about.
- Gemini's is `pickGeminiTitle` (`core/gemini-session.ts`) over the transcript path its context tail already tracks, including the `$set` history a resume replays, which is exactly the case the read leg exists for.
- Routing is not cosmetic: claude's resolver scans `~/.claude/projects` on a cache miss, so an unrouted grok/gemini node paid that scan every 60 s for a guaranteed null.
- **The sweep's gate lives in core, not in the shells.** `startSessionNameSweep` defaults `supports` to `supportsTitleRead` (`core/session-name-sweep.ts`) and neither shell passes it. The duplicated copies drifted, and reverting both to `canRename` left the whole suite green while silently skipping every gemini node.

### session to title (read, claude)

The authoritative name lives in the transcript `.jsonl`, not the OSC terminal title (`/rename` does not update OSC, a known Claude gap, so reading the file is the only thing that works after a resume).
- `core/transcript-reader.ts` `readSessionName(sessionId)` resolves the session file strictly by sessionId (no cwd fallback, which would make every Claude node in one folder resolve to the same newest transcript and adopt each other's names). `pickSessionName` returns the latest `custom-title`'s `customTitle` (the `/rename` name), else the latest `ai-title`'s `aiTitle` (auto name). Exposed over `pty.readSessionName`.
- `TerminalNode` polls it (about 4 s) only once this node's own sessionId is known and while the title still auto-tracks (`data.titleAuto`, default true on agent nodes), and adopts it as the `title`. `term.onTitleChange` feeds the `session` chip only.
- **A poll of an unchanged transcript reads no bytes.** The local read is gated on the resolved path's (size, mtime) (`titleCache`, bounded at 500, a failed tail read never cached). The remote one (`main/remote-title-reader.ts`) is gated on the remote context tail's `offsetFor` for the same path: that offset is the file size at the tail's last read, so a remote `/rename` lands within the tail's idle backoff (real gaps between idle reads about 3/5/9/11 s) plus one title poll (4 to 15 s). An untracked session (offset unknown) always reads.

### title to session (write)

The moment the user renames the node by hand (header rename box, ✦ AI-name, sidebar, command palette; all funnel through `applyManualTitle` or `renameSession`), `titleAuto` flips to false (polling stops overwriting) and the chosen name is pushed into the live session as `/rename <name>` via `pty.sendText` (tmux `send-keys`, the same one-way bridge as Branch's `/branch`; works whether or not the node is mounted). The launch command is left bare (no `-n`): Claude's own name is canonical until the user overrides it. `titleAuto` is persisted so an overridden name survives reload and resume.

## Search

The command palette (⌘K) matches the session name, tags and `nt-<id>` in the hint, and substring-searches each terminal's visible buffer (captured via `pty.capture` on palette open, cached about 3 s). Content matches show "found in output".

## Rules for UI that reads agent status

- A card chip that reads agent state subscribes to a **primitive signature** of the nodes it shows (`teamProgressSig`, `issueRunChipSig`), never to the whole `agentStatus.byId` map, which changes on every hook event of every node.
- A field of `agentStatus` that is persisted across a restart is restored as a record of the past, never as live state: `lastSeen` orders and ages sidebar rows but never becomes `state` or Eco's idle clock (`lastEventAt`).
- Before a card shows a fact, check that its place on the board does not already say it (`lib/cardRedundancy.ts`); the card modal keeps every fact the card drops.
- `project.ropes` and `bridges` are hostile input like the board: they are admitted through `sanitizeLinks` on every load and save seam, and a reader still tolerates anything. A wait rope (`--after`, the verify panel) is minted with `waitRopeId` (`ctrl-after-<dep>-<node>`), never with the opener's `ctrl-<source>-<node>` shape: rope order does not survive the canvas pruning ropes to deleted nodes, so the id is the only thing that tells a wait from an opener. A node opened by a control verb also records its opener (`data.openedBy`); readers that ask "who opened this" (`stationsByOpener`, the station-failure notice) prefer that record and still require the opener's rope to exist. In the renderer that rule is one function, `openerByTarget` (`lib/teamProgress.ts`): team progress and Tidy canvas both read it, and so must anything new that asks "who opened this node", passing the rope `id` through, or every wait reads as an opener.
- **Read agent status through the node's session** (`useSessionStores()` / `agentStatusForProject`), never `useAgentStatus` directly, in any surface that can show a team tab's node. A team tab keeps its host's status in its own session store and shares node ids with the SSH project it was handed off from, so the default store answers for the wrong project. See "A relay session owns its agent status" in [hosted-team-relay.md](hosted-team-relay.md).
