# Agent pause and hibernation

How restart, Eco hibernation, manual pause and bulk pause stop an agent CLI without losing the terminal session. Core paths: `terminal/agent-restart.ts`, `terminal/hibernation-policy.ts`, `lib/hibernationCandidates.ts`, `lib/bulkPause.ts`, `terminal/headless-pause.ts`.

## Restart agent (resume)

Restart is deliberately NOT a terminal-session lifecycle event: `terminal/agent-restart.ts` restarts the agent CLI inside the pane and leaves the PTY, tmux session and scrollback untouched. It picks up newly released models without manually closing and resuming every agent node.

- Quit with fixed Ctrl-C counts: `CTRL_C_QUITS` is codex 2, claude 3, grok 3, opencode 2, copilot 2. In those five, Ctrl-U clears only the cursor's line, so a typed exit submitted the rest of a multi-line draft; Ctrl-C clears the whole composer and quits once empty (#842, #928). Gemini alone gets bare typed `/quit`, because it was never measured and its composer is reachable only after login.
- `EXIT_SEQUENCES` remains the gate. An agent not in it can never restart in place.
- Poll `pty:pane-command` (`#{pane_current_command}`, local tmux socket or project SSH ControlMaster; a failure means "not a shell yet") every `RESTART_POLL_MS` (250 ms) until a SHELL owns the pane, then echo-deliver `resumeCommand(...)` (`claude --resume` / `codex resume`).
- **Nothing is killed.** If the CLI has not quit within `RESTART_EXIT_TIMEOUT_MS` (6 s), report `exit-timeout` and leave the session running. A user-asked restart adds `RESTART_LATE_EXIT_MS` (60 s) while the pane still reads: a 19 h cron session quit just after 6 s and left a bare shell with no agent and no resume (issue #899). Eco and Pause keep the bare 6 s. `exitTimeoutNotice` gives the bare resume line for that late-quit case.
- Refuse a `working` or `blocked` session: `/exit` typed into a permission prompt would ANSWER it, not quit. Hold one restart per node until the resume line has actually LEFT the pane, otherwise a second `/exit` can splice into an unsubmitted line.
- Bulk restart runs the same per-node closure sequentially over idle agents in canvas order and reports one summary line.
- `performRestartResume` composes `performExitPhase` plus `performResumePhase` (behavior-pinned split, 2026-08-12). Hibernation drives the halves separately; each refuses independently.

## Eco hibernation

Eco is opt-in and off by default: `settings.agentHibernationEnabled`, with `agentHibernationIdleMinutes` defaulting to 30 (Settings, Agents). Canvas runs a 60 s sweep and exits the CLI of at most **2** nodes per pass when they are hook-idle (`done`), fully offscreen (`isNodeWatched`, an open kanban card modal counts as watched), local, idle for the window, non-recurring and without live subagents. `planHibernation` and `lib/hibernationCandidates.ts` are pure and tested.

Tmux and the shell survive. The node shows a clickable SLEEPING chip. Wake (view, chip, modal open) verifies a SHELL owns the pane (`isShellCommand` or the persisted `hibernatedPane` that exit settled on, for xonsh/elvish users) before the KILL_LINE'd, echo-verified `withPermissionMode(resumeCommand(...))`. Sweep, wake and menu restart share one `guardConcurrentRestart` set.

- **The recurring fact is durable.** Both loop-card dismiss surfaces route through `lib/loopCard.ts`, which hides a cron/schedule card but retains `agentStatus.loop` (`dismissed: true`). Clearing it would let Eco `/exit` a CLI whose cron wakeup lives in that process.
- **Re-ask at fire time:** still-offscreen, remote and eligibility. A plan-time verdict is stale in seconds.
- `hibernated` self-heals on live hook states plus SessionStart, never on `done` (a late Stop POST must not undo a hibernate). Cold restore (`fresh`) clears it unconditionally and normally lets auto-resume own the node. `paused` is the deliberate exception: the flag clears, only relaunch is skipped.
- **Ordering with offscreen release:** Eco defers Phase-2 viewer release until the node hibernates (hard cap idle plus offscreen), but ONLY when the idle clock is known (`idleKnown`). `lastEventAt` is transient, so after an app restart nothing can hibernate and deferring would make Eco a memory regression. Eco is inert for sessions with no turn in the current app run. Persisted `agentStatus.lastSeen` is deliberately never idle proof for Eco, see "A restored clock is not an idle proof" in `terminal/hibernation-policy.ts`; user-confirmed bulk pause is the one place it is weak evidence.
- The deferral does not know `paused`: a deep-paused node's freshly recycled shell keeps its xterm alive until the hard cap, waiting for a hibernation that can never happen. This is a documented follow-up.
- The eight-item device checklist in PR #130 is required before recommending Eco.

## Pause session

Manual pause, or Eco with `settings.agentHibernationPersistAcrossRestart`, sets the persisted `agentStatus.paused` flag alongside `hibernated`. It has one job: stop a node from coming back on its own.

Two depths are chosen per node:

- **Shallow:** identical to an Eco exit (`registerAgentPause`'s `pause` closure reuses `performExitPhase`), then `paused`.
- **Pause and end session:** the same `restartAgentNode(…, restartShell: true)` recycle (`transport.recycle` plus `respawnNonce` bump), so the node next comes back fresh with no live tmux session holding memory.

`shouldColdResume` and `shouldAutoWake` in `terminal/hibernation-policy.ts` pin the contract. A fresh mount must not auto-relaunch a paused node. Mount timer, visibility-edge and kanban-card-modal-open auto-wake triggers must not fire for a paused node, hibernated or not. Only explicit Resume wakes, through the SAME `wakeHibernatedNode` trigger the SLEEPING/PAUSED chip uses, with the same `WakeInputBuffer` splice protection and retry budget.

Pausing an already-hibernated node skips the exit phase (`alreadyExited`): asking an idle CLI-less shell to quit would type `/exit` as a real command. `paused` is excluded from Eco's candidate plan and fire-time re-ask (`HibernationCandidate.paused`, `hibernationCandidates.ts`). A deep-paused node has `hibernated` unset because tmux was recycled, so `!hibernated` alone would admit it to a sweep after a dropped SessionEnd and stale `done`, the same `/exit`-into-a-bare-shell error `alreadyExited` closes manually.

Current UI: canvas right-click and sessions-sidebar row menus through the shared `selectionItems` builder; a read-only kanban card badge and a clickable card-modal badge; no command-palette entry.

## Bulk pause

Bulk pause is shallow pause applied to many nodes. It never recycles a tmux session and works for nodes that are NOT mounted (a project off screen has no `TerminalNode` pause closure). Surfaces: tab menu and sessions-sidebar project menu (*Pause project*, *Cancel pending pauses*, *Resume all*), group frame menu (*Pause group*), multi-selection of at least two agents (*Pause N sessions*) and a tab count badge.

1. **One primitive types by node id:** `PtyManager.sendPaneKeys(persistKey, data)` (`pty:send-pane-keys`, HOST_ONLY, `EDITOR_ONLY` in relay access policy and `SCOPED_REFUSED` for a scoped guest). Keys go over stdin to `tmux source-file -` through `controlInputPlan` / `remoteControlInputPlan`, never argv, to exact `=nt-<id>:`.
2. **An SSH-project node is never read or typed into on the local socket.** `paneRemoteRef` resolves an unmounted SSH node through `remoteNodeOwner` (machine-local index). A disconnected master or stray LIVE local session for an SSH-project node is `not-connected` / unreachable, never local fallback. `paneCommand` / `paneOwner` take opt-in `{ resolveUnmounted: true }`; the default stays legacy live-only because agent messaging and `terminateForeground` call them, and resolving an unmounted remote node could prove an owner on the host then act locally. Zellij, Windows session host, native Windows pane and plain shell answer `unsupported-backend`.
3. **`planBulkPause` (`lib/bulkPause.ts`) is the whole decision.** `now`: idle `done`, or unknown state with persisted `agentStatus.lastSeen.state === 'done'` (`idleForPause`). `whenDone`: working, waiting, blocked or held question/approval ticket, because `/exit` would answer a prompt. `alreadyExited`: Eco hibernated, so only mark paused. Named skips: `not-agent`, `no-session-id`, `not-resumable` (including an agent with no `exitSequence`, such as antigravity), `already-paused`, `relay`, `dropped`, `ended`, `state-unknown`. State is transient after restart, so an idle CLI and a mid-turn one look the same; only `lastSeen` survives. A skipped node says why. `bulkPauseResultText` counts every outcome (paused, pending, unreachable, unverified, not eligible, still running, skipped), so a disconnected-host batch never reads "Paused 0".
4. **`whenDone` is transient** (`state/pendingPause.ts`), never persisted because after restart nothing is watched. It fires on a CLEAN `done` edge only (not errored, interrupted or held), removes its mark BEFORE firing, and Resume, Cancel, node deletion or firing clear it. The watcher drops a mark when the node is paused elsewhere or status disappears. `BulkPauseTarget.sessionId` binds a mark to the planned session: the watcher drops it on session change, runner refuses a mismatch, and every Canvas funnel replacing the CLI (restart, fresh shell, model switch, reopen as, Claude/Codex account switches, bulk idle restart and both `transport.recycle` sites) removes it, otherwise a mark for one turn could quit the new session on its first done.
5. **Every exit holds `guardConcurrentRestart(nodeId, …)`** and one `createHostLimiter(BULK_PAUSE_HOST_CAP = 3)` per host key, shared by batch and pending fires. Owner reads are bounded. The key is `local` or SSH project id, not host, because ControlMasters are per project. `pauseNodeHeadless` (`terminal/headless-pause.ts`) runs `performExitPhase` through `sendPaneKeys`, with the menu restart's late window (`RESTART_LATE_EXIT_MS`); a failed key write ends that wait at once (`unreachable`). A pane owner not readable before exit is `unverified`, distinct from `not-eligible`; neither types anything.
6. **Resume all is active-project only.** Waking is mounted `wakeHibernatedNode`, so elsewhere the row is disabled with "Open the project to resume".

Desktop supports local and SSH. Server Edition has the core primitive and real ws-bridge leg, but pending fires need an open browser tab because the watcher is renderer-side. Relay tabs are refused. Kanban card menus inherit the rows through `nodeActionItems`. Mobile is a follow-up.

## Pty persistence and environment

The node id is the `persistKey` passed to `transport.create`, so it must stay stable. If tmux is unavailable, `PtyManager` falls back to a plain shell with no cross-restart continuity.

`findTmux()` resolves an absolute path because GUI apps do not inherit shell PATH. Its order is fixed system paths, then the shell's PATH, then the tmux the macOS app ships (`bundledTmuxPath`). System first is deliberate: a machine with tmux keeps its own, so the bundled copy is a floor, never an override. `resourcesPath` is undefined on Server Edition, so its bundled binary is unreachable by construction; a Linux host is expected to have its own. Under `electron-vite dev` the last candidate resolves against `process.cwd()`, where `scripts/build-tmux.mjs` writes its artifact.

If all candidates miss, the fallback strips `TMUX` / `TMUX_PANE` from child env to avoid nesting refusal. The same strip, one layer up, applies to a launcher that was itself an agent session: started from a Claude Code terminal, the app used to hand `CLAUDE_CODE_CHILD_SESSION` and siblings to every pane, and the Claude CLI then turned transcript saving off (no Cmd+M view, context meter or `--resume`). `buildPtyEnv` deletes explicit measured `NESTED_AGENT_ENV_STRIP` names (`core/nested-agent-env.ts`), and `ACCOUNT_SCOPE_UPDATE_ENV` lists the same names so a tmux server seeded from such env does not hand them back. Explicit names only, never a `CLAUDE*` prefix sweep: `CLAUDE_CONFIG_DIR` and `CODEX_HOME` select managed accounts and must survive.
