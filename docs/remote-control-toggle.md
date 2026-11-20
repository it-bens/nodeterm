# Claude Remote Control toggle

An antenna button for agents in `REMOTE_CONTROL_CAPABLE` (claude only), in the terminal header right of the context meter and in the kanban card modal. Both use one component, `components/RemoteControlButton.tsx`. Green means this session is reachable from claude.ai or the Claude app; a click types Claude's own `/remote-control` into the session.

It replaced a short-lived app-wide toolbar toggle for the phone relay, which read as unrelated to any terminal.

## State is read from the transcript, because no hook exists

`core/remote-control-scan.ts` is fed by both context tails (local and SSH) and broadcasts `agent:remote-control` into `state/remoteControl.ts`, a transient per-node store stamped with the session it was read from.

- ON is a `system` / `bridge_status` record. OFF is a `local_command` record whose stdout says "Remote Control disconnected". The latest record wins.
- **Only the `local_command` shape is measured** (Claude Code 2.1.325). The other two strings come from the CLI's source, and no live connect has been run against the parser. It therefore matches exactly and degrades to OFF: it can under-report, never invent ON.
- A node's state counts only for its current session (`remoteControlOn`), so a node that moves to another conversation does not inherit the old one's state.

## One command, two different dialogs

`/remote-control` is one toggle (`renderer/lib/remoteControlToggle.ts`).

- OFF to ON may show a one-time "Enable Remote Control?" consent (measured). We never answer it.
- ON to OFF opens a dialog whose first row is "Disconnect this session". We press Enter **only after reading the pane and finding the cursor on that exact row** (`disconnectRowSelected`). A timer-based Enter would accept the consent dialog whenever our state was stale.
- The card modal has no live write, so there the disconnect dialog is left for the user.

## When it may type

- Refused while `blocked` or `waiting` (`remoteControlRefusal`): typed text answers a prompt. Allowed while `working`, because Claude runs the command immediately, mid-turn.
- Hidden on relay tabs (no local tail).

## Surfaces

- Desktop: full.
- Server Edition: works (same core scan, a real bridge subscription).
- Mobile: N/A.

## Device checks owed

- A real connect turns it green.
- A real disconnect turns it off, and the gated Enter fires.
- The state after an app restart and after `--resume`.
