# tmux session continuity

How each terminal lives inside a persistent tmux session, and the rules for every tmux call the app makes (local and over SSH). Main code paths: `src/core/pty-manager.ts`, `src/core/tmux-naming.ts`, `src/shared/ssh.ts`, `src/core/da-reply-gate.ts`. Lifecycle by intent (park, release, reopen, cold restore) is in [terminal lifecycle](terminal-lifecycle.md); SSH specifics are in [remote sessions](remote-sessions.md); emulator-side behavior is in [terminal emulation](terminal-emulation.md).

## The tmux model

`PtyManager` runs each terminal inside a persistent tmux session (`tmux new-session -A -D -s nt-<nodeId>`) on a dedicated socket (`-L node-terminal`) with a generated config (`-f <userData>/tmux.conf`, so the user's `~/.tmux.conf` never interferes). The config has the status bar off, **mouse on**, 50k history, `set-clipboard on` plus `terminal-features ",*:clipboard"`, and the copy-mode mouse bindings. Because the tmux server outlives the app, sessions survive when no client is attached. `remoteTmuxConf` in `src/shared/ssh.ts` is the same config for an SSH project's remote tmux.

If tmux is unavailable, `PtyManager` falls back to a plain shell (no cross-restart continuity). The node id is the `persistKey` passed to `transport.create`, so it must stay stable.

A node id never reaches tmux as a bare `-t` (see Exact targets below), and the local tmux client is never spawned from the node's directory (see Client cwd below).

## Exact tmux targets

Every node id that becomes a `-t` goes through `exactPaneTarget` (`=name:`) or `exactSessionTarget` (`=name`) from `core/tmux-naming.ts` (security audit F1/F2, 2026-10). Both helpers refuse a name `sessionName` did not produce.

- tmux resolves a BARE target by exact name, then fnmatch, then PREFIX on a miss, and node ids end in a counter or a random tail. With only `nt-term-abc-12` alive, `send-keys -t nt-term-abc-1` typed into it, `capture-pane` read it and `display-message '#{pane_pid}'` named its process (measured, tmux 3.4).
- That made every node-keyed verb (send-text, typed chat input, send-keys, capture, pane command/cwd, pane owner, terminate-foreground, has-session, the control-mode shadow, an attach-only reattach) a way to reach a DIFFERENT node, whether its own session was gone or the id was a relay peer's invention. A relay peer naming a prefix typed into, read and killed another project's terminal.
- For an existing session `=name:` is the same pane the bare name was (current window, active pane, re-measured with three windows and a split).
- An exact miss is `can't find session` (exit 1), except `display-message`, which exits 0 with every format EMPTY. Every parser here answers null for that.
- On a REMOTE command line the target is single-quoted: zsh's EQUALS option expands a word starting with `=`.
- Proven against a real tmux driving a real `PtyManager` (`pty-exact-target.realtmux.test.ts`).
- A new tmux call site owes the same.

## Synchronous tmux calls are bounded

Every synchronous tmux call on main is bounded (`TMUX_BOOT_CMD_TIMEOUT_MS`, 2 s, via `PtyManager.runTmuxSync`). Never use a bare `execFileSync` on the tmux socket.

- `ensureTmux`'s `source-file` runs before the first window exists, and `ensureUpdateEnvKeys`' `show-options` / `set-option` run on a spawn. Unbounded, a server that accepts the connection and never answers held main for the wedge's whole lifetime (4.05 s in a repro, i.e. forever against a server that never recovers).
- The bound kills only our short-lived client (SIGKILL, so one that ignores SIGTERM cannot keep `execFileSync` waiting), never the server.
- A timeout is `console.warn`ed and fails open (tmux stays resolved; the spawn goes ahead). "No server yet" stays silent.
- A timeout also ends the rest of that call's tmux work (the next `set-option` would only wait out the same wedge). A `show-options` / `set-option` timeout starts `TMUX_UPDATE_ENV_COOLDOWN_MS` (60 s) in which `ensureUpdateEnvKeys` makes no tmux call at all: every spawn passes the account-scope names, so without the cooldown each later spawn on a slow server would stall main again (20 terminals is about 40 s). After the cooldown the next spawn probes again.
- Proven against a real client in `pty-tmux-wedged.realtmux.test.ts`.

## A spawn-env write does not reach a tmux session on its own

The shared tmux server takes each new session's env from its own GLOBAL env (inherited from whichever client started the server). The creating client's process env only matters for names listed in `update-environment` (or passed as non-secret `-e` pairs). Setting `env.FOO` in `pty-manager` therefore works for the plain-shell fallback and for the one client that happens to start the server, and silently does nothing (or leaks the server-starter's value into everyone else) after that. That is how issue #419 shipped: managed-account `CLAUDE_CONFIG_DIR` leaked into system-account sessions.

New per-session env either joins `ACCOUNT_SCOPE_UPDATE_ENV` / the gateway list, or rides `-e`, and gets a real-tmux test (`account-env.realtmux.test.ts` is the pattern).

## Remote tmux invocations

Every REMOTE tmux invocation starts with `remoteTmuxPathPrologue()` (PATH append, so a non-login ssh exec channel finds tmux). The reasoning and the tmux-less degrade are in [remote sessions](remote-sessions.md).

## tmux owns the mouse, scrolling and the alternate screen

This is the native behavior, and it is deliberate.

- **The wheel scrolls tmux's own history** (`history-limit`), not the emulator's buffer.
- **The pane is on the alternate screen** (`\e[?1049h`); capabilities are NOT blanked. That is what keeps a full-screen TUI's input box in place instead of scrolling away with the text.
- **Selection is tmux copy-mode.** A drag copies. Apps that request mouse tracking themselves (vim, htop) still get their own mouse events, because tmux forwards those regardless.

**Do not take scrolling away from tmux again.** A previous design did exactly that (`mouse off` plus `terminal-overrides ',*:smcup@:rmcup@:indn@'` to keep tmux on the normal screen, so its output flowed into xterm's scrollback, which was then hydrated from `tmux capture-pane` on reattach). It failed structurally: **tmux is a screen PAINTER, not a stream.** Every redraw (attach, resize, refresh) erases and repaints, so blank and duplicated rows leaked into the emulator's scrollback. Users saw black bands and duplicated screens when scrolling up, and the pane stopped behaving natively. The hydration that design needed is gone (see reattach seeding in [terminal emulation](terminal-emulation.md)).

## Copy goes to the system clipboard via OSC 52

`set-clipboard on` plus `set -as terminal-features ",*:clipboard"`: on copy, tmux emits OSC 52 to the attached client, and the renderer's OSC 52 handler (`parseOsc52` in `terminal/osc52.ts`, applied in `TerminalNode.tsx`) writes the system clipboard. Two traps, both measured on tmux 3.4:

- **The `terminal-overrides ',xterm*:Ms=…'` entry does NOT work on tmux 3.2+.** With it, a copy emitted zero OSC 52 to the client. `terminal-features` is what actually enables the sequence. Do not "fix" the `Ms=` override back; it is why copying from SSH sessions never worked.
- **No `pbcopy` pipe.** The copy-mode bindings are bare `copy-pipe-and-cancel` (no command). Piping to `pbcopy` was macOS-only, and over SSH it would have copied on the remote host anyway. OSC 52 is cross-platform and works over SSH.

### Copy-on-select

Opt-in (`copyOnSelect`, default off, #759). It is the route for a selection xterm owns, which OSC 52 never sees: a plain drag on Windows (no tmux), a forced Option/Shift drag inside a mouse-tracking app. `terminal/copy-on-select.ts` triggers on the GESTURE (a press xterm's SelectionService takes, then a release anywhere), not on `onSelectionChange`, because the search addon's `select()` must never touch the clipboard. It applies to the canvas node and the kanban modal only, never the settings preview. It is attached once per xterm (it survives park/adopt) and written via the bridge's `{ quiet: true }` path so a failed write raises no toast per drag.

## Pty input is filtered for exactly one thing: duplicate DA answers

Every viewer answers the tmux client's handshake; only one answer may reach it (issue #1172).

- A new tmux client queries the terminal it is drawn in (DA1 `ESC[c`, DA2 `ESC[>c`, XTVERSION, OSC 10/11, measured on 3.4). With N viewers on one pty (two Server Edition tabs, a canvas node plus its kanban card modal, a relay peer) all N xterms answer. tmux consumes ONE DA answer inside its ~5 s query timeout and passes the rest, and a lone late one, to the pane as typed input. So `1;2c0;276;0c` landed in front of agent launch lines (the echo check passes: the line itself is intact) and in live prompts on every reattach. OSC replies are always consumed by tmux.
- The gate is `core/da-reply-gate.ts`, per Session, fed from `proc.onData` and applied in `PtyManager.write`. A write made ONLY of DA1/DA2 replies spends one open query of its kind per reply, and a reply with no open query (a duplicate, an unsolicited one, or one past `DA_REPLY_TTL_TMUX_MS` = 4 s for a real tmux client) is dropped.
- Rules: match only `CSI ? … c` / `CSI > … c` (no key produces them, while a broader report match eats keystrokes: `ESC[1;2R` is both CPR and Shift+F3). Touch only whole reply-only writes (xterm.js emits each reply as its own data event, so a mixed write is a person). Never close the gate for the session's life (a pane app's DCS-passthrough query shows up as the same `ESC[c` and gets exactly one answer).
- Do not widen the gate to "anything that looks like a terminal report": a filter that eats a keystroke is worse than the bug.

## A tmux client is not necessarily a watcher

`SessionInfo.clients` is a COUNT (`#{session_attached}`), never a boolean, because one session can hold several clients: the app's painter, the user's own `tmux -L node-terminal attach`, a second nodeterm on the same socket, and our own **control-mode shadows** (`PtyManager.shadowAttach`, used for background writes without spawning a painter). The session reaper subtracts ours via the `shadowed` seam: a shadow is a real client but not a watcher, so a shadowed session must stay exactly as cullable as an idle detached one.

The count is carried numerically rather than collapsed at parse time because the subtraction needs it. A session holding our shadow AND a real client must still read as attached, and a boolean could only be forced to false, reaping the session out from under whoever that other client belongs to. Any future reader of `list-clients` / `session_attached` owes the same subtraction.

## Client cwd: the local tmux client never runs in the node's directory

The local tmux CLIENT is spawned from `localTmuxClientCwd()` (home, else `/`), never the node's cwd. The pane's directory rides `new-session -c` only (issue #1131).

- Any client can be the one that starts the shared server, the server keeps that client's cwd for life, and from tmux 3.7 `spawn.c` applies a pane's `-c` only when the server's own `getcwd()` succeeds. Once the directory the server started in is deleted (a removed worktree), every NEW pane on the server inherits the deleted dir whatever `-c` said, and agents launched there hang.
- Measured on a locally built tmux 3.7c (private socket): client in the doomed dir gives a `-c <proj>` pane at `<gone> (deleted)`; client in `$HOME` gives a pane at `<proj>`. tmux 3.4 to 3.6 honour `-c` either way, which is why CI cannot reproduce it and the guard is a spawn-options assertion (`pty-tmux-client-cwd.test.ts`).
- The plain-shell fallback is the pane itself and keeps the project cwd. Remote tmux is started from an ssh exec channel, which sshd places in the login home.
- A server started by an earlier build keeps its old cwd until it is restarted (`tmux -L node-terminal kill-server`, which ends every session).

## Ending a session and remote ownership

**User clicks ×** runs `destroy(persistKey)`, which runs `tmux kill-session`, permanently ending the session. For a REMOTE node it kills the remote session and then the local one of the same name. The local kill is normally a no-op, but it reaps the orphan the pre-`requireRemote` local fallback could leave behind.

**A node's OWNERSHIP is persisted state, never a live object.** "Is this node remote / whose host is it on?" must be answerable with nothing attached, because the questions that ask it (a delete, a kill, a cleanup) arrive precisely when nothing is: after an app restart, after the offscreen release, after the park timer, for a project that is not open.

- `runEndSession` used to read it off the dying `Session` alone (`dying?.sshRemote`), and its comment claimed "both callers run while the session is still live", which is false for the case that matters. `dying` was then `undefined`, remoteness read as "local", the remote branch was skipped in silence, and the one kill that went out went to the LOCAL socket, where a `requireRemote` node has nothing at all. Everything else about the teardown ran, so the node left the canvas looking deleted while its `nt-<id>` kept running on the host, a leak with no surface anywhere.
- The durable answer is the machine-local index (`core/remote-end.ts`, `planRemoteEnd`): the shell wires `PtyManager.setRemoteNodeOwner` to `workspaceStore.sshProjectIdForNode` plus that project's ControlMaster (`refForProject`). A LIVE `sshRemote` still wins when there is one, because it is the exact master the session was spawned over and a node created seconds ago may not be in the index cache yet. The two sources are complementary, not redundant: treat a live handle as the complement of the index answer, not its source.
- The Server Edition wires no resolver (it has no SSH-project manager) and its deletes stay on the local path unchanged.

**The kill is CHECKED.** The old `catch {}` read "remote session may not exist / master down; ignore", which are not the same fact: tmux's own exit 1 ("can't find session", `probeSaysAbsent`) is an ANSWER, while ssh's 255, a 127 or a spawn error is a NON-answer with the session still running.

- A non-answer, and a node whose project has no master at all, is **recorded** (`core/pending-remote-kills.ts`, atomic JSON under userData, keyed by `user@host` because several projects share one host's tmux server) and settled the next time that host connects (`SshProjectManager.settleOwedKills`, hung on the shared connect attempt so the REUSE branch pays too; an entry is dropped only on tmux exit 0 or 1).
- The delete itself is **never refused** over an unreachable host: the node is going, and a refusal strands it on the canvas with the same session still running plus a dialog, which the user answers by deleting it again. That trade is only defensible BECAUSE the debt is durable; drop the store and refusing becomes the honest option.
- **Only a `delete` may owe a debt.** A `recycle` keeps the node (worktree move, model switch, "pause and end session"), so a kill deferred to a later reconnect would land on the session that node has since RESPAWNED under the same name, ending live work hours after the action that queued it. A recycle still ATTEMPTS the remote kill; it just records nothing when it cannot land.

## A remote node is never spawned locally

`PtyCreateOptions.requireRemote`: `sshRemote` says "here is the master to run over"; `requireRemote` says "and if there isn't one, spawn NOTHING".

Without it, a create with no `sshRemote` falls through to core's local tmux/plain-shell branches. That is how an SSH project's terminal opened while the ControlMaster was down (no network, host unreachable, `ssh` missing) quietly became a LOCAL shell in the local `$HOME`: same node id, same `SSH user@host` header chip, the REMOTE session's scrollback snapshot replayed into it, and for an agent node a cold-restore `claude --resume <remote session id>` running on the wrong machine under the local account, leaving an orphaned local `nt-<id>` behind.

- Refused on both sides: the renderer never calls `create` when `resolveSshRemote` came back empty (`CoState.offline` plus the node's Reconnect button), and core refuses in `spawnNew` (`PtyCreateResult.unavailable`) so a master that dies inside the round trip cannot sneak through.
- `join` refuses (`'held-elsewhere'`) a `requireRemote` create whose held session is local or on another host (host, user, or port differs). A join to the SAME remote's live session (a second view over the same endpoint) still attaches, and so does a create that names no `sshRemote` onto a held remote session (its master momentarily down, or a relay tab's create on the host, which carries none). The held session is never killed.
- A relay / team tab's node, which may carry no `sshRemoteTmux` at all, is kept off this core by the session it runs under plus a renderer-side `spawnBelongsElsewhere` refusal (see [projects and persistence](projects-and-tabs.md)).
- An offline node reports itself to `SshReconnector`, so the canvas heals itself. `retryNow` (banner Reconnect / node Reconnect) skips the backoff and clears the refuse window.

**Whether a node is remote comes from its PROJECT, never from its own fields** (issue #1186, `@shared/ssh-project-nodes`). `requireRemote` / `sshRemoteTmux` used to be stamped only by this machine's node factories, but an SSH project's `project.json` lives on the host and other writers put nodes into it, above all the host's own desktop, which opens the same folder as a LOCAL project (`cwd: "."`, no `ssh` fields). Such a node ran on the VIEWING machine in an unrelated folder, under the id the host was running. Three layers:

1. `fileToProject` routes every session node of an SSH project to the project's host (`routeSshProjectNodes`: `ssh` = the project server, `sshRemoteTmux`, cwd resolved POSIX-style against `remoteCwd`; editor/video/files get `sshFs`), and `persistedCanvases` / `getNodeResolved` read the cache the same way.
2. `projectToFile` writes the SSH mirror NEUTRAL (`neutralSshProjectNodes` drops those fields, cwd back to `./x`), because the file is shared layout: a node stamped "runs over ssh on <host>" cannot run on the host's own desktop at all.
3. Core refuses a local spawn whose `ownerProjectId` is an SSH project (`PtyManager.setSshProjectCheck`, wired in BOTH shells from the index) even without `requireRemote`.

Kept as written: a plain `ssh <host>` terminal (`ssh` without `sshRemoteTmux`, `shell: 'ssh'` at the spawn) and a foreign host ATTACHMENT (`sshRemoteTmux` plus another host). The one local terminal an SSH canvas may hold is a LOCAL-account login node, marked in memory only (`renderer/lib/localOnSshCanvas.ts`, `localOnSshCanvas` on the create; a scoped relay guest's is deleted). After a restart it routes to the host like any other node. **Downgrade:** a build older than this one reads the neutral file's nodes as local (the bug it already had for host-made nodes).

### The phone's relay `pty.attach` is closed the same way

`PtyManager.prepareRelayAttach`, with pure routing in `core/relay-attach-plan.ts`. It used to call `attachDetached(nodeId)` (`tmux new-session -A` on the LOCAL socket) for whatever id the phone named, with no `sshRemote` and no `requireRemote`. An SSH project's node opened on the phone before the desktop mounted it became a local shell in this machine's `$HOME` (no context meter, the markdown chat view fallback, work on the wrong machine), and the desktop's later mount created a SECOND `nt-<id>` on the host: one node id, two sessions, two machines. Rules a refactor must not undo:

- **Where is decided from THIS machine's records, never the phone's words.** The desktop wires `setRelayNodeResolver` (`workspaceStore.relayNodePlacements`, which covers EVERY project holding the id, with the node as that project recorded it, plus `SshProjectManager.spawnRefFor` and the index's SSH identity). A node is remote when it carries `sshRemoteTmux` + `ssh` (a host attachment routes over its attachment scope, as `sshConnectionScope` does) or sits in an SSH project. A plain `ssh` terminal (`ssh` without `sshRemoteTmux`) runs `ssh` locally, as the renderer does. The phone's optional `projectId` may only CHOOSE among those placements, or REFUSE (an unknown id it places in an SSH project); it never routes a node anywhere on its own.
- **Remote = over the project's live master, with `requireRemote`, or refused** with a sentence the phone shows (`{message, reason}`: `not-connected` / `no-ssh` / `still-connecting` / `unregistered-remote`). Never a local fallback. A master whose connect setup has not finished (`Conn.setupDone`, set exactly where `connected` is emitted) may only JOIN a session the host positively lists (the renderer's `waitForSshRemote` rule): a session created before the setup chain carries no hook/account env and no tmux.conf, for life.
- **The env is the desktop's, from the one builder.** The plan hands `spawnSession` the node's recorded `agentId` / `agentModel` / `accountId` / `cwd` / `shell` / `ownerProjectId`, so `buildPtyEnv` (agent id, canvas-control grant, permission wait), the account dir and project overrides apply exactly as on a desktop create. No second env is written for the relay. The shared pre-spawn refusals (`requireRemote`, the managed-Codex scope) are ONE helper, `spawnRefusal`, used by `spawnNew` and the relay. Pane ownership is still NOT recorded for a relay-created session: the owner would be derived from project files, which is the one source that ledger refuses to trust.
- **One id in a local AND an SSH project** (the same committed canvas opened in two folders) is answered by whichever session exists: a live LOCAL one wins, otherwise the remote one.
- **Existing damage is reported, never killed.** A stray local `nt-<id>` the old path left for a node that only lives remotely is never attached from the relay again (and the desktop's own mounts already pass `requireRemote`), and a relay attach that finds one logs a `[relay] … left untouched` warning. It is not killed automatically: it may hold work typed into it, and nothing proves it idle. Do NOT point a user at the session-memory panel's × for it: a delete resolves remoteness from the index (`planRemoteEnd`) and would end the REAL remote session too. The manual cleanup is `tmux -L node-terminal kill-session -t =nt-<id>`.
- Server Edition: no phone relay host, no resolver wired; `prepareRelayAttach` there is the bare local attach it always was.
- Tests: `relay-attach-plan.test.ts` (matrix), `pty-relay-attach.test.ts` (real PtyManager, node-pty mocked), `remote-security.test.ts` (host reply), `main/relay-attach-wiring.test.ts` (the desktop wire, at source level).

## One tmux socket per host

An SSH project spawns on the host's `node-terminal` socket, the same one a nodeterm running on that host uses. See [remote sessions](remote-sessions.md) for the full rule (`RMT_TMUX_SOCKET === TMUX_SOCKET`, pinned by `src/core/remote-ssh/unified-socket.test.ts`; `nodeterm-rmt` is the legacy socket: never spawned on, but a session an older build left there is adopted in place, so a remote tmux command about one session goes through `sessionTmuxCmd` and names the socket as `SESSION_SOCKET_REF`).

## Session-host geometry follows the most recently active viewer

Like tmux's `window-size latest` (issue #914). Under tmux a relay-mirrored phone is its own tmux client, so dismissing its keyboard gives it its rows back. On the session host (Windows, no tmux) the phone and the desktop node share ONE client socket, and the old componentwise minimum held the phone to the desktop node's rows with an empty band and no explanation.

`latestClaimSize` (`core/pty-size.ts`) picks the claim with the highest recency, bumped by an attach, a claim that CHANGES, and a write that is not an emulator's automatic answer (`core/terminal-reports.ts`; every attached xterm answers a DA/CPR/OSC query, and counting those would hand the session to whoever answered last). Three rules, each load-bearing:

1. A viewer that cannot adapt is a CEILING. A pty wider than the phone's screen would wrap into garbage there (or, rendered at the pty's size, be clipped to its left ~45 columns), so a relay sink is `bounding` unless `pty.attach` said `resizedFrames: true`. The iOS app deliberately does NOT send it: it reads `OP.Resized` only to show a "Sized to another screen · Fit this screen" hint, and every phone report is ANSWERED (the sink's `sinkShown` is forgotten on each report) because the phone clears that hint whenever it sends a size.
2. The real size flows back to every viewer (`SessionHostPty.onSize` to `PtyManager.applyBackendSize` to `pty:size`, and `OP.Resized` to the sink), and a session-host `Session`'s `applySize` only VOTES.
3. The host's `geometry` push is NEGOTIATED at `hello` (`SESSION_HOST_FEATURES`) and sent only to sockets that asked. An older client reads every non-`data` push frame as an EXIT.

Across app connections the host still takes the minimum. Full write-up: [Windows session host](windows-session-host.md), "Output ordering, flow ownership, and geometry".
