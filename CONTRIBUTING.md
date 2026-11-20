# Contributing to nodeterm

nodeterm puts live terminals and agent sessions on a pan-and-zoom canvas. Contributions are accepted under [BUSL-1.1](LICENSE); see the license for production-use terms and the change to MIT. Fixes, tests, and refactors of license checks for development are welcome.

## Getting set up

Use a current Node.js 22.x release. CI and `.nvmrc` select the 22.x line. `package.json` supports Node.js `^22.22.2 || ^24.15.0 || >=26.0.0` for this checkout. Use that range for both Desktop and Server Edition source builds, even where an installer or bootstrap checks a lower minimum.

On macOS (12 Monterey or newer) or Linux, run these from the repository root:

```sh
npm install
npm run dev
```

On Windows, run `bootstrap-windows.bat` instead of `npm install`, then `npm run dev`. The bootstrap checks for Python 3 and the Visual Studio C++ build tools, including the x64 Spectre libraries, before running `npm ci`. Choose a supported Node.js version before running it.

Install tmux if you want POSIX terminal sessions to outlive the app. For native module repair use `npm run rebuild`, which patches the dependencies before compiling them. See [Packaging and updates](docs/packaging-and-updates.md#development-commands) for build, preview, and platform-specific setup, including the GCC 14+ workaround.

For the browser UI, run `npm run server:dev` on Linux or macOS. See [Server Edition Quickstart](docs/SERVER.md#quickstart) for host setup and how to restart without rebuilding.

## Where code goes

| Directory | Owns |
| --- | --- |
| `src/core/` | Shared services. Use `CorePlatform` for shell-provided operations. |
| `src/main/` | Electron shell: windows, native dialogs, and desktop IPC wiring. |
| `src/server/` | Server Edition shell: HTTP and WebSocket RPC for the browser UI. |
| `src/preload/` | Desktop bridge: exposes `window.nodeTerminal` through Electron's `contextBridge`. |
| `src/renderer/` | React UI. Calls core through the session's API; does not import the shells. `src/renderer/bridge/` supplies browser and relay implementations. |
| `src/shared/` | Shared types and logic. Define IPC channel names in `src/shared/ipc.ts`. |
| `src/session-host/` | Standalone Windows terminal session host. |

Add shared service logic to `src/core/`, not `src/main/`. Neither `src/core/` nor `src/server/` may import Electron or `src/main/`. Inject shell-specific dependencies, and decide what happens when a shell cannot provide them. Add terminal session operations through `TerminalTransport`; do not reach around it from a node or card.

## Surfaces

- **Desktop:** macOS, Linux, and Windows (beta). Keep new code platform-neutral. Use the platform that owns the filesystem when resolving paths.
- **Server Edition:** the same renderer in a browser, backed by the server's core. A new `window.nodeTerminal` member needs a real bridge implementation or an explicit, documented refusal. Type checking cannot distinguish a working bridge from a no-op stub.
- **Mobile companion:** separate iOS and Android repositories. Name any follow-up for relay, pairing, or session data in the PR and tag `@eneskirca` when a mobile change is needed. Do not assume every phone is an iPhone.

The canvas and kanban board are two views of the same nodes. When changing a node action, badge, or terminal interaction, check the canvas, board card, and card modal in the same change. State which surfaces are unaffected or unsupported and why.

## House rules

- Check `src/shared/` and the owning subsystem for an existing rule or helper before adding one. Keep identifiers, comments, and UI copy in English.
- Write comments that explain why the code is needed and name the failure it prevents. Do not restate the code.
- Use `thisMachine()`, `thisMachineCap()`, or `machineNoun()` from `src/renderer/lib/machineName.ts` in user-visible copy. A Server Edition tab uses neutral wording because it describes the server, not the viewer's machine.
- Write `\x00` instead of a raw NUL byte in source so Git keeps the file reviewable as text. See `src/shared/source-hygiene.test.ts`.
- Attach an `'error'` listener to a child's stdin before writing. A child that exits before reading can emit `EPIPE` asynchronously; `try/catch` does not catch it. Use the child's exit status to decide success. See `src/core/stream-epipe.guard.test.ts`.
- Writes to `process.stdout` or `process.stderr` can also emit an asynchronous `'error'`, not just throw. Where a stream can close, listen for the event and stop forwarding to the failed stream. See `installLogSink` in `src/core/log-sink.ts`.
- Use the atomic write helpers for file publication. See [Atomic writes](docs/atomic-writes.md) for local and SSH writers.
- Keep shared project content separate from machine-local settings and execution consent. See [Workspace persistence](docs/workspace-persistence.md#rules-for-changing-persistence-code).
- Before changing behavior, read the owning topic below. Update that topic with any changed contract and keep cross-references current. Put measurements and unverified cases where the next contributor can find them.

## Testing

```sh
npm run typecheck
npm test
```

Run both before opening a PR. Test behavior with inputs that distinguish the fix from the bug. For a new guard, remove or invert it locally and confirm the test fails. Use source scans for structural boundaries, not as substitutes for exercising behavior.

Keep tests in the suite's isolated home, temp dir, and tmux sockets. Preserve the sandbox when starting subprocesses. Gate platform-specific tests with a reason instead of letting them fail on other platforms. If a test reads a fixture or source file and depends on line breaks, normalize CRLF before matching. See [Test sandboxes](docs/test-sandboxes.md) for helpers, cleanup, and test-writing habits.

Report any device or environment checks you did not run. For performance changes, measure the real behavior before and after on the same setup; name the build and what was measured. See [Performance](docs/performance.md).

## Pull requests

- Branch from `main`. Check the `quality`, `CodeQL`, and `Dependency review` results.
- Lead with what changed and why it matters. Then say how you checked it, what you did not verify, and any remaining risks or follow-up.
- Keep the description proportional to the change. Include before-and-after numbers and explain rejected alternatives when they matter to the decision.
- If using an agent to write PR text, apply the `pr-writing` skill.

## Topic map

These pages own the subsystem rules, source paths, and device checks. [README](README.md) covers the product and installation; [CLAUDE.md](CLAUDE.md) is the short pointer for agents.

### Workspace, canvas, and board

- State and projects: [Workspace persistence](docs/workspace-persistence.md), [Projects and tabs](docs/projects-and-tabs.md), [Project limit](docs/project-limit.md), [Open recent](docs/open-recent.md), [Open links](docs/open-link.md).
- Canvas tools: [Camera and focus](docs/canvas-camera.md), [Layouts and actions](docs/canvas-layout-actions.md), [Panels](docs/canvas-panels.md), [Worktrees](docs/worktrees.md).
- Board and GitHub: [Kanban board](docs/kanban-board.md), [GitHub issue cards](docs/github-issues-kanban.md), [GitHub sync](docs/github-sync.md), [Pull request status](docs/github-pull-status.md), [Board dispatch](docs/board-dispatch.md).

### Terminals, nodes, and desktop UI

- Session continuity: [tmux](docs/tmux-continuity.md), [Terminal lifecycle](docs/terminal-lifecycle.md), [Session backends](docs/session-backends.md), [Windows session host](docs/windows-session-host.md), [Session memory](docs/session-memory.md).
- Node views: [Node kinds](docs/node-kinds.md), [Terminal node](docs/terminal-node.md), [Terminal emulation](docs/terminal-emulation.md), [Browser node](docs/browser-node.md), [Run node](docs/run-node.md), [Simulator node](docs/simulator-node.md), [Whiteboard](docs/whiteboard-node.md), [Trigger node](docs/trigger-node.md), [Dev-server ports](docs/dev-ports.md).
- UI: [Appearance](docs/appearance.md), [Node icons](docs/node-icons.md), [Keybindings](docs/keybindings.md), [Window chrome](docs/window-chrome.md), [Notch HUD](docs/notch-hud.md), [Speech and dictation](docs/speech.md).

### Agents, hooks, and orchestration

- Agent support: [Adding an agent](docs/adding-an-agent.md), [Antigravity](docs/antigravity-agent.md), [Copilot](docs/copilot-agent.md), [Cursor](docs/cursor-agent.md), [Gemini](docs/gemini-agent.md), [Grok](docs/grok-agent.md).
- Integration: [Hook model](docs/hook-model.md), [Integration consent](docs/agent-integration-consent.md), [Node identity](docs/node-identity.md), [Shared Codex node identity](docs/shared-codex-node-identity.md), [Hook-reply approvals](docs/hook-reply-approvals.md), [Claude turn interrupts](docs/claude-turn-interrupt.md).
- Accounts and live state: [Managed accounts](docs/managed-accounts.md), [Model gateway](docs/model-gateway.md), [Permission mode](docs/permission-mode.md), [Usage indicator](docs/usage-indicator.md), [SSH Codex metrics](docs/codex-ssh-metrics.md), [Context meter](docs/context-meter.md), [Notifications and status](docs/notifications-and-status.md), [Subagent cards](docs/subagent-cards.md), [Pause and hibernation](docs/agent-pause-hibernation.md).
- Collaboration: [Chat view](docs/chat-view.md), [Context Link](docs/context-link.md), [Agent messaging](docs/agent-messaging.md), [Canvas control](docs/canvas-control.md), [Launch dependencies](docs/launch-dependencies.md), [Durable orchestration state](docs/durable-orchestration-state.md).

### SSH, relay, mobile, and Server Edition

- SSH and sharing: [Remote sessions](docs/remote-sessions.md), [Native SSH on Windows](docs/native-ssh-windows.md), [SSH agent skills](docs/ssh-agent-skills.md), [Live links](docs/live-links.md), [Team presence](docs/team-presence.md), [Hosted team relay](docs/hosted-team-relay.md).
- Mobile: [Phone relay](docs/phone-relay.md), [iOS relay protocol migration](docs/ios-protocol-migration.md), [Mobile chat](docs/mobile-chat-view.md), [Mobile usage inbox](docs/mobile-usage-inbox.md), [Push webhooks](docs/push-webhook.md).
- Host setup and operations: [Server Edition](docs/SERVER.md).

### Builds, updates, performance, and tests

- Build and maintenance: [Packaging and updates](docs/packaging-and-updates.md), [Uninstall](docs/uninstall.md), [Atomic writes](docs/atomic-writes.md), [Test sandboxes](docs/test-sandboxes.md).
- Measurement: [Performance](docs/performance.md), [Idle energy](docs/idle-energy.md).
