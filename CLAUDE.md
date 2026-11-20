# nodeterm: agent notes

Start with [CONTRIBUTING.md](CONTRIBUTING.md) for setup and the topic map.

## Before editing

- Do not remove or bypass license, entitlement, or usage-limit checks for production use ([LICENSE](LICENSE) §Additional Use Grant).
- Put shared service logic behind `CorePlatform`; keep Electron behind the shell and PTY traffic behind `TerminalTransport` ([CONTRIBUTING.md](CONTRIBUTING.md#where-code-goes) §Where code goes).
- Check Desktop, Server Edition, and mobile; check both the canvas and the board for node changes ([CONTRIBUTING.md](CONTRIBUTING.md#surfaces) §Surfaces).
- Treat shared project files as untrusted; keep `shell`, `ssh.extraArgs`, and `pendingLaunch` machine-local ([docs/workspace-persistence.md](docs/workspace-persistence.md#rules-for-changing-persistence-code) §Rules for changing persistence code).
- Never spawn an SSH project's remote node locally when its host is unreachable ([docs/tmux-continuity.md](docs/tmux-continuity.md#a-remote-node-is-never-spawned-locally) §A remote node is never spawned locally).
- Distinguish releasing a terminal viewer from ending a session; check whether the backend preserves live work ([docs/terminal-lifecycle.md](docs/terminal-lifecycle.md#lifecycle-by-intent) §Lifecycle by intent).
- Keep tests off live tmux sockets and out of the real home and agent config ([docs/test-sandboxes.md](docs/test-sandboxes.md#the-suite-never-touches-a-live-tmux-server) §The suite never touches a live tmux server; §The suite never touches the real home).

## Navigation

- Workspace state, projects, canvas, and board: [CONTRIBUTING.md §Topic map](CONTRIBUTING.md#workspace-canvas-and-board).
- Terminal backends, node views, and desktop UI: [CONTRIBUTING.md §Topic map](CONTRIBUTING.md#terminals-nodes-and-desktop-ui).
- Agents, hooks, chat, and orchestration: [CONTRIBUTING.md §Topic map](CONTRIBUTING.md#agents-hooks-and-orchestration).
- SSH, relay, mobile, and Server Edition: [CONTRIBUTING.md §Topic map](CONTRIBUTING.md#ssh-relay-mobile-and-server-edition).
- Builds, updates, performance, and tests: [CONTRIBUTING.md §Topic map](CONTRIBUTING.md#builds-updates-performance-and-tests).
