# Terminal lifecycle

What happens to a terminal node's xterm and PTY client across unmount, park, offscreen release, window hide, app quit, reopen, reboot (cold restore) and focus changes. Main code paths: `src/renderer/nodes/TerminalNode.tsx` (the trickiest file), `terminal/park-budget.ts`, `terminal/offscreen-policy.ts`, `terminal/webgl-budget.ts`, `terminal/live-work.ts`, `core/scrollback-store.ts`. The tmux side (sessions, kill, remote ownership) is in [tmux continuity](tmux-continuity.md); the emulator side (reattach seeding, widths) is in [terminal emulation](terminal-emulation.md); pause and Eco are in [agent pause and hibernation](agent-pause-hibernation.md).

## Lifecycle by intent

### Node unmount (project switch): the renderer parks the terminal

`TerminalNode.tsx` `parkedTerminals`: the xterm instance and its attached PTY stay alive with the `.xterm` element detached from the DOM, so a remount within the park window re-adopts them, instantly and exactly. The tmux client never detaches, so mouse-tracking and alternate-screen modes and scrollback carry over. Do NOT "optimize" this into a respawn plus redraw: a fresh xterm on a reused client misses the attach-time mode sequences and breaks scrolling. The park timer then runs the real teardown: `kill()` detaches the PTY client and the tmux session keeps running.

- **Window and cap are settings (issue #886).** `settings.terminalParkMinutes` (default 10, raised from 5; **0 = until app quit**: `parkWindowMs` returns `null` and NO timer is armed, never `Infinity`, which `setTimeout` clamps to about 1 ms and would dispose every park at once) and `settings.terminalParkMax` (default 20, raised from 12). Both are re-validated at park time.
- The LRU cap evicts **local parks before remote ones** (`planParkEviction`'s `isRemote`). A local re-adopt miss is a warm reattach in milliseconds. An SSH one is a new client per terminal paced 4-per-master, plus a login if the master idled out (`ControlPersist`), and a live parked client is also what keeps that master from idling out.
- Parking does not raise sshd `MaxSessions` pressure: masters are per project, so a parked SSH project holds exactly the channels it held while in view.
- Measured (headless xterm, 200x50): a parked tmux-backed terminal is about 2 MB (alternate screen, empty normal buffer, tmux owns the history); a plain shell with a full 10k scrollback is about 27 MB.
- Permanent-delete paths call `disposeTerminalOnUnmount(id)` so a deleted node disposes instead of parking.

### Offscreen release (in place)

A mounted node fully offscreen past `settings.offscreenTerminalMinutes` (default 10, `0` = never; Settings, tmux) detaches its PTY client and disposes its xterm without unmounting. A plate is shown, tmux keeps running, and reattach-redraw on approach was measured under 500 ms. The released node is a DETACHED tmux session, so it joins the session reaper's candidate pool (the 6 h grace still protects it). See "The terminal component" below for the two invariants (mount-stable observer; `session.source` remote gate).

### A hidden window counts as offscreen

Close-to-tray (the Windows default), macOS hide-on-close and minimize send `app:window-visibility` (`main/window-visibility.ts`). The renderer flag (`terminal/window-backgrounded.ts`) makes every node report itself offscreen through the SAME mount-stable observer path, so the release runs at `min(offscreen setting, BACKGROUND_RELEASE_DWELL_MS = 2 min)` behind every existing gate (live-work, held launch, Eco, remote, selected, `0 = never`). After that dwell Canvas runs the memory-pressure reclaim pair (`runReclaimLevers`), which now frees the on-screen terminals' WebGL contexts too because they all read hidden. Show cancels the dwell and revives warm.

- It feeds existing levers; it is not a new one. Reuse that input rather than adding a parallel release path that would have to re-implement every gate.
- An occluded window or a Server Edition tab switch never triggers it (explicit IPC, not `visibilitychange`; the bridge declares it a no-op). Eco and the liveness poll see a hidden window as "nobody watching" too.
- **A raw Electron `hide` is NOT the signal.** On macOS `BrowserWindow` emits `hide`/`show` from occlusion changes (a window merely covered by another app, another Space, Cmd+H), so main forwards `hide` only when the close branch set `windowHiddenByApp` before its own `win.hide()`. `minimize` always counts. A focused renderer clears the flag (fail-safe for a window manager that restores without a `restore` event).

### Every memory lever asks whether the kill ends live work

`terminal/live-work.ts`. The renderer reclaims terminal memory in FOUR places: park window expiry, the park's LRU cap, the memory-pressure drop (all three in `park-budget.ts`) and the offscreen viewer release (`offscreen-policy.ts`). All four were written as if dropping a PTY client were free, because "the tmux session keeps running and re-attach redraws". **That sentence is only true where tmux is actually underneath.** On the plain-shell fallback (no tmux installed, tmux switched off in settings, or an install path `findTmux` missed) the pty IS the shell, so the identical call kills it and everything under it, an agent CLI mid-turn included. Issue #126: a project switch terminated a working Claude agent, which then auto-resumed from wherever the kill landed.

- The predicate is deliberately the narrowest one that closes it: a tmux-backed session is never protected (the kill costs a redraw), and neither is a plain terminal.
- **An IDLE agent CLI on a non-persistent pty IS protected** (`agentProcess`, `agentProcessInPane`; not once hibernated, paused, dropped, or once its CLI announced a SessionEnd, `sessionEnded`, which is its own transient flag because `state: undefined` alone is also what an idle agent looks like). Killing it looked free because cold restore `--resume`s it on revive, but the resumed CLI fires `SessionStart:resume` and idles with no further hook event, the status mirror leaves it unverified, and agent messaging refused it for as long as it stayed idle (measured 2026-09-13 on Windows native ptys). The mirror now also lets a verified `idle_prompt` right after a verified `SessionStart` commit a verified non-inferred `done` (`MirrorEntry.sessionStarted`), and the decider reports a proven node reset by a boundary as `targetNotIdleUnknown`, not `targetStatusStale`.
- **A fifth lever owes the same gate:** the offscreen release of an ARMED node (`shouldDeferReleaseForHeldLaunch`). Held launches require the attached transport for echo verification, on every backend. Keep that transport until delivery or recovery; a blind paste by session name loses the shell-init repair and canonical-line protections.

### WebGL contexts are viewport-scoped and budgeted

Browsers cap about 16 live contexts, and a canvas holds far more terminals. A per-terminal `IntersectionObserver` (`rootMargin` pre-announces approach) only REPORTS visibility to a module-level budget coordinator (`terminal/webgl-budget.ts`) that owns every grant decision and all timing. It keeps the contexts WE hold at or under the live budget so the browser never has to **force-evict**, which is the bug that flashed Chromium's dead "lost context" placeholder (white box plus sad face) on a visible terminal during a fast pan or zoom out, because the old per-node observers each acquired independently and momentarily overshot the cap.

- Budget: `WEBGL_BUDGET` 12 by default (the browser Server Edition). On DESKTOP main raises Chromium's cap itself via `--max-active-webgl-contexts` = 32 and boot raises the budget to 24 via `setWebglBudget` (constants in `src/shared/webgl.ts`).
- A client is granted only after an **acquire debounce** (`WEBGL_ACQUIRE_DEBOUNCE_MS`, so a pan-through never grabs a context for a two-frame flash).
- If granting would exceed the budget, **reclaim on demand from the least-recently-visible HIDDEN holder** (`hiddenAt` LRU order). If every holder is currently visible (zoomed way out), the newcomer is NOT granted and **stays on the DOM renderer**; we never push past the budget.
- A hidden holder keeps its context **indefinitely** (warm for a pan-back of any length). There is no time-based release; it is reclaimed strictly on demand, either by a visible newcomer that needs its slot or by `releaseAllHiddenGrants` (queued through the drain) under memory pressure.
- `acquire()` returning false (WebGL2 unavailable) does not burn a slot.
- An externally-lost context (`onContextLoss`) is reported via `handle.contextLost()`, drops from the accounting, and, for a still-VISIBLE client, schedules ONE delayed budget-gated re-grant (sleep/wake GPU resets lose every context at once with no visibility change; without this every woken terminal sat on the DOM renderer until panned out and back). The NODE still never re-acquires itself (that loop is the eviction fight the design fears): the retry goes through `tryGrant`, never exceeds the budget, never reclaims a visible holder, and stops after `WEBGL_LOSS_STREAK_MAX` consecutive losses (a visibility transition resets).
- The node registers via `registerWebglClient` on mount and `handle.dispose()`s on unmount (which releases and cancels timers). A parked terminal is off-screen so it holds no context.
- **A dead or hung process is logged, never acted on.** `child-process-gone`, window `unresponsive`/`responsive` and external WebGL context losses each write one tagged line to the debug log ring (`main/process-health-log.ts`, `[webgl]` in `terminal/webgl-budget.ts`) and nothing else. Chromium respawns its own GPU process; a reload or relaunch on top of that turns a GPU reset into a lost session. If you add a loss path, report it through `handle.contextLost(kind)` so the external counts stay apart from our own `loseWebglContexts` releases.

**A renderer released while the node is unmeasurable mismeasures its own row spacing** (`terminal/dom-renderer-spacing.ts`). `WebglAddon.dispose()` is also the back-to-DOM-renderer path, and it runs from the lifecycle effect's CLEANUP, after React detached the element. The fresh DOM renderer then derives `letter-spacing` from a width cache whose `offsetWidth` is 0 and bakes in a whole extra cell per character. That is the "letters drift apart for a split second after a project switch": the adopting mount paints wide until the WebGL grant lands 150 ms later. Focus mode's `display:none` wrapper is the same shape. xterm re-derives the spacing on a char-size / dpr / options change and not on a resize, so nothing in the reattach path heals it. `applyFit` calls the change-gated `resyncDomRendererSpacing(term)`, which bails while the measurement is still 0 rather than re-baking the wrong number.

**Which renderer a terminal uses** is `settings.terminalGpuRendering`, resolved by the single resolver `resolveTerminalRenderer(value)` (`src/shared/webgl.ts`) to `dom | webgl | shared`:

- `'off'` = xterm's DOM renderer.
- `'on'` = one budgeted WebGL context per terminal (everything above).
- `'shared'` = glyphgrid, ONE canvas-wide WebGL2 context every terminal paints into (`src/renderer/glyphgrid/`, reached through `terminal/glyphgrid-attach.ts`; the per-terminal budget is OFF in this mode).
- `'auto'` (the default, and what legacy/unknown values fall back to) = `webgl` on EVERY platform, macOS included. The macOS branch has moved twice: it was `dom`, then `shared` on 2026-08-05 (per-terminal WebGL composited terminals black after zoom-out bursts, blamed on the OS compositor), and is now `webgl`. The blackout was root-caused not to context count but to a dependency skew (addon-webgl 0.19's dispose crashed on the 5.5 core and aborted its own DOM-renderer restore; pinned and healed, see `renderer/terminal/webgl-addon-pair.test.ts`). What actually guards macOS is a lower budget, `WEBGL_BUDGET_DESKTOP_MAC` (16, vs 24 elsewhere), capping compositor pressure at every zoom.
- The four-way setting stays as the escape hatch: `'shared'` is opt-in only (also where the macOS default points back if the one unconfirmed 2026-07-30 whole-window-flicker report recurs), and `'off'` drops GPU rendering entirely.

### Window close and app quit

Clients detach (`PtyManager.killAll()`); the tmux session keeps running. `killAll()` deliberately does NOT kill sessions.

**Window close off macOS = background mode** (`settings.runInBackgroundOnClose`, `'auto'` = on for Windows, off for Linux; `resolveBackgroundOnClose` in `main/background-mode.ts`). The x HIDES the window into a notification-area tray (`main/tray.ts`, pure model `tray-model.ts`: Open / unread / phone-access status / Quit) so the phone relay host, hook server, notifications and updater keep running (the Dock analogue).

- A hide is not a quit, so no confirm dialog. Tray **Quit** is plain `app.quit()` (before-quit and confirm run as for the menu Quit: File, Exit on Windows, where Electron's quit role has NO Ctrl+Q accelerator). Never set `quitting` before it; a cancelled confirm must leave the x hiding.
- Hide only while the tray actually EXISTS (`backgroundOnCloseActive`), so a failed tray never turns x into a hide with no way back.
- The updater and update-prep quits keep flipping `quitting` first, or a hidden window would block them.
- The first hide posts a one-time toast (userData marker file, not a settings key, because the renderer's full-snapshot saves would overwrite a main-side settings write).

### Node reopen or app relaunch (nothing parked)

A new PTY attaches to the same `nt-<nodeId>` session and tmux redraws current state. Ending a session (the x button) and the remote-ownership rules are in [tmux continuity](tmux-continuity.md).

## Cold restore (machine reboot)

tmux only survives an **app** restart; a **machine reboot kills the tmux server**, so every `nt-<nodeId>` session is gone. To bridge that, `create()` returns `PtyCreateResult` with a `fresh` flag. It asks the tmux server whether the session already exists BEFORE spawning, so `fresh=false` means a warm reattach (tmux redraws) and `fresh=true` means a cold start (first open or post-reboot).

Locally that ask is one `tmux has-session` per node. **On an SSH project it is ONE `tmux list-sessions` per host per burst** (`core/remote-ssh/remote-session-index.ts`), because a project switch mounts every node in the same tick and N probe channels on top of N pty channels overrun a stock host's `MaxSessions 10`. The measurement and the failure chain it closes are in that module's header. The two rules a refactor must not undo:

1. Only tmux's own exit 1 is evidence of absence. Every other outcome answers "exists", because a transport failure read as "cold" replays a snapshot and types `claude --resume …` into a LIVE agent pane.
2. A session this process just spawned is recorded (`markPresent`) and a remote kill invalidates the cached list, so nothing inside the cache window can be told it is cold when it is not.

On a cold start the renderer (`TerminalNode.tsx`) reconstructs state instead of relying on the dead session (you cannot keep a live OS process across a reboot). See [remote sessions](remote-sessions.md) for the late cold-start check that second-guesses an unknown verdict.

### Scrollback replay

`core/scrollback-store.ts` keeps a byte-capped (256 KB) snapshot of each tmux session's recent output under `<userData>/terminal-scrollback/`, refreshed on a timer (`SCROLLBACK_SNAPSHOT_MS`) and on detach/quit (`tmux capture-pane -e`). On a cold start the renderer reads it via `pty.readScrollback` and writes it back into xterm (with a "session restored" separator). Warm reattach skips it (tmux already redraws). It is deleted with the node in `destroySession`.

**The periodic capture is paced, serialized and deduplicated** (`snapshotTick`, `core/scrollback-cadence.ts`). A working agent's spinner keeps its session dirty forever, so the old tick spawned a `capture-pane -S -1500` (an ssh exec for a remote node) and rewrote up to 256 KB for EVERY busy session, all in the same instant, every 15 s. Now:

- A dirty session is captured on its first `BUSY_AFTER_TICKS` (4) consecutive dirty ticks, then every `BUSY_EVERY_TICKS`-th (60 s). An off-cadence tick KEEPS the dirty bit, which is what lets detach/quit still take their final capture; an idle tick resets the count.
- Captures run one at a time on `snapshotChain`, and a session whose capture is still queued is never queued twice (`snapshotQueued`).
- A failed capture OR a failed disk write re-marks it dirty (`writeScrollbackIfChanged` reports the write; a skipped unchanged capture counts as done).
- Every write (periodic, detach, quit) goes through `writeScrollbackIfChanged`, which skips a capture whose sha1 matches the last one written. The digest is dropped with the file in `endSession`, so a recreated node always writes.
- The cost is bounded and deliberate: a continuously busy session's post-reboot replay can be up to about 60 s stale, since the snapshot only serves a machine reboot.

### Agent resume

On a cold start of a node whose `agentId` is in `RESUMABLE_AGENTS`, the renderer re-launches the agent CLI: `resumeCommand(agentId, sessionId)` (from the session id persisted in `agentStatus` localStorage: `claude --resume`, `codex resume`, `gemini --resume`) when known, else the bare `launchCmd`. The one-shot `data.initialCommand` still wins on the very first open, so the agent is never double-launched.

- **The one exception is a `paused` node** (see [agent pause and hibernation](agent-pause-hibernation.md)): it skips this auto-relaunch, which is the entire point of the flag, and instead records the pane its fresh shell settled on (`agentStatus.hibernatedPane`), so a later explicit Resume can recognize it even for a default shell outside the wake's `isShellCommand` allowlist.
- An armed node (`pendingLaunch`) is also refused the cold relaunch; see [launch dependencies](launch-dependencies.md).

**A persisted session id is not evidence the conversation still exists**, and a dead one is not a no-op: `claude --resume <dead id>` prints "No conversation found with session ID: <uuid>" and EXITS, leaving the pane at a bare shell under a node still wearing its agent badge. Measured 2026-09-09 on the reporting host (`nodeterm-rmt`, 108 live sessions): **20** panes sat in exactly that state, and not one of those 20 ids had a `<id>.jsonl` anywhere under the system `~/.claude/projects` or the managed account root. Claude's own 30-day cleanup, a `/clear`, a removed account and an id minted for a session that never ran all produce it.

So the cold-restore branch asks `chat.transcriptExists` first (`transcript:exists`, served by `registerTranscriptIpc` in BOTH shells) and, on a POSITIVE `absent`, launches the agent **bare** and raises a slim `CoState.lostSession` banner on the node ("The previous conversation could not be found, this agent started fresh.") instead of opening a blank session in silence. Three rules make that safe:

- **The answer is a TRI-state** (`TranscriptPresence`: `present | absent | unknown`) and only `absent` drops the id. The two errors are wildly asymmetric: wrongly resuming a dead id costs one line and a bare shell, wrongly dropping a LIVE id strands work the user believes is continuing. So an unreadable root, a `readdir` that threw, a downed ControlMaster, a relay tab, a rejected call and an id we would not put on a command line are ALL `unknown` = resume exactly as before. `transcriptPresence` (`core/transcript-reader.ts`) is `resolveTranscriptPath` plus that one distinction, not a second way of finding a transcript, and a root that does not exist at all still answers `unknown` on purpose: cold restore runs at boot, where a not-yet-mounted `$HOME` is indistinguishable from an empty one.
- **The remote leg's `unknown` is TERMINAL, never a fallthrough.** A remote session's transcript is on the host, so searching this machine for it would find nothing and report `absent` about the wrong computer. `remoteTranscriptPresence` (`src/main`) exists beside `remoteTranscriptRefFor` rather than reusing it because that function collapses "not remote" / "no home" / "ssh failed" / "the host looked and there is nothing" into one `undefined`, which is fine for a reader that falls back and fatal for a caller that acts on absence. `locateRemoteTranscriptCommand` was already built for this distinction (it exits 0 on a clean miss, "so no transcript is an ANSWER, not a failed ssh"), and that is the property this reads. It is deliberately more permissive than the local leg (it searches the account root AND the system root), which errs toward `present` = keep the resume.
- **Claude only**, gated by `readsClaudeTranscript`. A codex/gemini/grok id misses this resolver by construction every time, so probing one would answer `absent` for a healthy session and drop its resume. Those agents keep the pre-existing behavior until the probe learns their layouts (`handoff/locate.ts` has the per-agent locators; that is the seam).

Decision logic is the pure `terminal/cold-resume-session.ts`. **Adding a `CoState` field owes the hand-written equality list in `setCo`**: a patch touching only an uncompared field is SWALLOWED and its banner silently never renders (it happened once to `spawnError`). `nodes/cold-resume-wiring.test.ts` asserts every field of the interface appears there.

Surfaces: Desktop full. Server Edition full (the handler is core, the ws-bridge leg is real, and that shell runs on the host whose transcripts it reads, so it needs no remote leg). Relay tabs answer `unknown` and resume as before. **Not yet on the kanban CARD MODAL:** `ModalTerminal` reads no `CoState`, so a user who only ever opens the session from the board does not see the notice. The honest fix is a shared node-notice surface rather than a second copy of the banner.

### The app's own environment is not the panes' environment

Whatever launched nodeterm (often an agent CLI's shell) leaves markers in `process.env` that describe THAT session. `buildPtyEnv` strips an explicit list (`TMUX`, `NESTED_AGENT_ENV_STRIP`). Add a name there, with its measurement, rather than a prefix sweep: `CLAUDE_CONFIG_DIR` and `CODEX_HOME` select managed accounts and must survive.

## The terminal component (gotchas)

`src/renderer/nodes/TerminalNode.tsx` is the trickiest file.

### Effects, identity and the released state

- The xterm instance and PTY session are created once in a `useEffect(…, [data.respawnNonce, offscreenEpoch])` and torn down on unmount. The component persists across re-renders because React Flow keys nodes by `id`: never change a node's id, or you will respawn its terminal.
- **Third in-place state, "released" (offscreen dispose):** a node fully offscreen in the canvas viewport for `settings.offscreenTerminalMinutes` has its xterm and PTY client torn down IN PLACE (node stays mounted showing a plate, tmux session untouched) and revives (warm reattach) when it re-approaches the viewport. Pure policy: `terminal/offscreen-policy.ts`. Two load-bearing rules a refactor must not undo:
  1. The **visibility IntersectionObserver lives in its own mount-stable `[termKey]` effect**, NOT the lifecycle effect. The down transition re-runs the lifecycle effect, and an observer owned there dies with it, making revive unreachable (permanent plate; caught in review). The lifecycle run publishes to it through refs (`visibilityReportRef`, `offscreenLiveRef`, identity-checked on clear).
  2. The remote exclusion asks `offscreenCoreIsRemote(session.source)` (`'local'` only is eligible; relay/server tabs excluded), NOT `data.remote`, which is a field nothing sets on node data (a gate on it was constant false and type-invisible; pinned by tests).
  SSH-project nodes are also excluded; collapsed = hidden (same convention as the WebGL budget); a `respawnNonce` bump while released revives first. Agent-status and fan-out clears live in a dedicated unmount-only effect (a release or respawn must not blank a live badge).
- **React StrictMode is deliberately not used** (`main.tsx`): double-mount would spawn two PTYs per node.

### Hover guard and pointer contract

The xterm container is `nodrag nowheel`. A transparent **hover-guard** overlay sits on top until you dwell `settings.panHoverDelay` (so quick drag = move node, scroll = pan). After the dwell the guard is removed and xterm takes input. The header stays draggable. The guard listens to POINTER events (`nodes/HoverGuard.tsx`): React Flow's d3-drag swallows a left `mousedown`/`mouseup` on a draggable node before React sees them, so the old mouse-event guard never received a left click (the click-to-focus of #87 only ever worked through the dwell).

### Sloppy focus (default)

`settings.terminalFocusFollowsPointer` ON. The dwell TAKES the keyboard, and `mouseleave` only re-arms the guard: it never blurs the xterm or drops the active flag / presence focus. The terminal keeps the keyboard until another node takes it (its dwell, a click) or the user clicks elsewhere. This replaced strict focus-follows-mouse, where leaving the node released everything and typing with the pointer resting on the canvas went nowhere. Release follows DOM focus through the SAME hook click-to-focus uses (below), so a dwell onto a node whose Cmd+M view covers its xterm blurs the OTHER terminal's xterm (`dwellBlursForeignTerminal`), else the previous pane keeps eating keystrokes while another node reads as active. The guard stays a POINTER contract in this mode: the hook's `setArmed` is a no-op here (`focusDrivesGuard`).

### Click to focus

`settings.terminalFocusFollowsPointer` OFF (issue #757, Settings, Behavior). The pointer decides nothing: no dwell, and `mouseleave` changes nothing. A click (`HoverGuard` pointer events to `onGuardClick` to `enterNow`) or a "go to node" takes the keyboard, and the node's active flag, presence focus AND guard then follow DOM focus. Only a literal `false` in settings.json selects it (`resolveFocusFollowsPointer`). The Cmd+/ shortcuts panel prints "Click" instead of the dwell. Renderer only: Desktop and Server Edition identical; kanban card modal N/A (it has no hover guard); Mobile N/A.

ONE hook owns all of it, `nodes/useClickToFocus.ts`, and it binds to the stable `.term-node` ROOT, never the React Flow wrapper. Focus mode MOVES that root into the fullscreen surface (`surface.appendChild(root)`), so a listener or containment check captured on the wrapper went deaf there and read every body press as an outside press. The wrapper is re-resolved at event time only to recognise the node's own React Flow chrome (resize handles).

- Root `focusin`/`focusout` run `focusLossOutcome` (`lib/terminalFocusMode.ts`): focus moving inside the node or the WINDOW blurring (Cmd+Tab) keeps it. A press on the node's own chrome (header drag; React Flow focuses its wrapper, MEASURED in Electron 42) hands it back to the element that lost it (the Cmd+M composer) or the xterm (`reclaimTarget`). Anything else (another node, a field, the empty canvas: `onPaneClick` blurs the xterm textarea, `shouldReleasePaneFocus`) releases it and re-arms the guard.
- A press on the Cmd+M chat's MESSAGE TEXT (`.term-chat__msgs`) is the start of a drag-selection, so its reclaim waits for the press to end (`reclaim-on-release` to `reclaimOnRelease`). A plain click still hands the keyboard back to the composer, but a press that selected text leaves the selection (and Cmd/Ctrl+C) alone. Reclaiming at once moved the selection into the textarea and nothing in the chat could be highlighted or copied.
- One document capture `pointerdown` does the rest: outside the node it releases activity claimed WITHOUT focus (go-to-node under the Cmd+M view, Canvas's own `setActive` on a jump, where no focusout ever comes, `outsidePressReleases`). A document capture `focusin` landing outside the node (its own wrapper counts as inside) does the same for KEYBOARD focus moves (Cmd+M open, then Cmd+K's autofocus), else the stale `activeId` suppresses that node's unread dot.
- Inside the BODY, any deliberate primary press that is not on the guard runs `enterNow` (`bodyPressAcknowledges`) (guard down, xterm focused, Cmd+M view open, all the same), so an unread finish is cleared by clicking the terminal, not only by clicking the guard. A focus RESTORE that no press caused (window activation) never acknowledges.
- The xterm blur that OPENING the Cmd+M view causes is `keep`, not a release (`lostIsCoveredXterm`).
- Focus mode's reparent blurs a focused xterm SYNCHRONOUSLY inside `appendChild` (MEASURED, Electron 42: `relatedTarget` null, root still connected, so an `isConnected` test cannot see it). `nodes/reparentKeepingFocus.ts` brackets the move with a flag the hook honours (`reparenting`) and re-focuses the element that held the keyboard, in BOTH modes. Before it, entering/leaving focus mode dropped the keyboard in the default mode too.

### Where the wheel stops being the terminal's

Decided by HIT TEST, per packet: `Canvas.tsx` answers `overNativeScrollable` with `target?.closest('.nowheel')`, and React Flow's own `panOnScroll` walks the same class (`noWheelClassName`). Issue #767 reported the second consequence below as the first.

- **(a) Inside the body the wheel is already the terminal's, band included.** `.term-node__xterm` carries `nowheel` AND is `position: absolute; inset: 0` over a body with no padding and no border, so the visible inset band (the host's own `4px 2px 6px 6px`) and a co-attach letterbox band are part of the HOST's hit area. MEASURED with `elementFromPoint` under headless Chromium against the verbatim rules, and pinned as a three-link CSS invariant by `canvas/terminal-wheel-boundary.test.ts`. The styles.css line the report reads ("insets the xterm by a few px") is about PAINT (the band shows the body's `--term-bg` because the host paints none), and paint is not hit testing. An overlay laid over a LIVE terminal therefore owes `pointer-events: none` (`.term-node__stalecwd`, joining `.term-node__upload` and `.term-copy-pill`). An overlay that REPLACES a dead view (`.term-node__offscreen`, `.term-node__closed`) deliberately keeps the canvas wheel: there is nothing underneath to scroll, so panning is the useful answer.
- **(b) The boundary that actually moves is TEMPORAL, not spatial.** While it is up, `.term-hover-guard` covers the whole body and is NOT `nowheel`, so for the first `panHoverDelay` after the pointer enters, and again after it leaves, a wheel over terminal TEXT pans the canvas. That is the guard's own contract ("quick drag = move node, scroll = pan canvas") and the reason those incidents cannot be reproduced on demand. Hoisting `nowheel` to `.term-node__body` would swallow the guard with it (a second consumer, React Flow's, reads the class the same way and no per-element opt-out can reach it). Hoisting it to the whole NODE would additionally take wheel-zoom-to-cursor away over every node, which is exactly where a `wheelZoom` user aims.

### Fit and resize

- **FitAddon reads the host's computed size, not its content rect.** The absolute, inset canvas host uses `box-sizing: content-box` so its padding is excluded from that size (#671). Its outer hit/plate rect still fills the body. The board modal instead keeps padding on a separate wrapper. Do not put border-box padding back on a fit host: it over-reports rows and clips the last line. `scripts/terminal-fit-layout.test.ts` measures real xterm layout through resize sweeps at DPR 1, 1.25, 1.5 and 2 in Chrome (`CHROME_BIN` overrides the executable); this does not verify GPU row-seam rendering.
- A `ResizeObserver` drives `FitAddon.fit()` plus `transport.resize`. Canvas zoom is a CSS transform, so it does NOT change `clientWidth`: cols/rows stay stable across zoom. `scale-fix.ts` patches xterm's mouse coords so text selection stays aligned when zoomed.
