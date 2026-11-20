# Permission mode and approval gates

The mode an agent session starts in (`--permission-mode`, `--approval-mode`, `--ask-for-approval`), how each agent's vocabulary is translated, the version gates, and the codex auto-reviewer NEEDS YOU gate. Main code: `src/shared/agents/approval-mode.ts`, `renderer/state/permissionMode.ts`, `core/codex-cli.ts`, `core/claude-cli.ts`, `core/agents/codex-permission-gate.ts`.

Related: [adding an agent](adding-an-agent.md) (items 8, 9, 13), [grok](grok-agent.md), [gemini](gemini-agent.md), [hook reply approvals](hook-reply-approvals.md).

## Membership and translation

Agents in `PERMISSION_MODE_CAPABLE` (claude, grok, gemini, codex) have a **start** mode (`claude --permission-mode <mode>`; Shift+Tab still cycles it at runtime). Membership no longer implies claude's flag spelling: the per-agent translation lives in `src/shared/agents/approval-mode.ts` (`approvalFlags` / `modeSupported`), which is also where `withPermissionMode` lives (it moved one layer up out of `config.ts` to break a cycle).

- gemini = `--approval-mode default|auto_edit|yolo|plan`.
- codex = `--ask-for-approval on-request|never` (and `untrusted` too, but only on a codex that still has it; see below).
- Two rules the mapping enforces:
  1. A mode the CLI **cannot express emits NO flag**, never a substituted nearest match. Codex has no `plan` and no edit-specific mode. **Gemini has no `auto`**: nothing in its vocabulary means "approve most things but not edits", and since `auto` is the DEFAULT mode, mapping it to `auto_edit` would have switched auto-approve-edits on for every existing gemini node at upgrade time, silently.
  2. "Supports" must not be a lie either: codex's `manual` maps to `untrusted` because its built-in default is `OnRequest` (measured: `codex doctor`, no `approval` key in `~/.codex/config.toml`), so leaving it unflagged would deliver "the model decides when to ask" under an "Ask each time" label. Codex is the first agent where `manual` emits a flag.
- The UI copy is DERIVED from the mapping (`permissionModeAgentIds` / `permissionModeAgentsLabel` / `unsupportedModesNote` / `bypassSandboxCaveat`) so a sentence cannot drift from what the table does. The note reads "Auto has no Gemini equivalent…" beside codex's gaps; the residual wart is only that `auto` and `manual` land on the same gemini policy (the prompting one).
- `--sandbox` is a separate axis and deliberately untouched (`--ask-for-approval never` still sandboxes).

## An agent's vocabulary is not a constant: the codex probe (#785)

Measured release by release on the published linux-x64 binaries: 0.146.0, 0.147.0 and 0.148.0 advertise `untrusted, on-request, never`; **0.149.0** through 0.154.0 advertise `on-request, never`. clap does not ignore a value it does not know, it prints `error: invalid value 'untrusted'` and **exits**, so a Manual-mode Codex node launched from the pinned table died at the prompt and left the pane at a bare shell.

- The gate is codex's OWN and sits beside claude's, never inside it (a gate fed by `claude --version` belongs to claude).
- `core/codex-cli.ts` reads `codex --help` once per app run, FEATURE-detected rather than version-compared (a floor guesses about builds that are not on npm), parses the option's own slice with `codexApprovalValuesFrom` (the neighbouring `-s, --sandbox` carries its own `[possible values: …]` two lines up, and a page-wide scan emits `--ask-for-approval read-only`), and publishes `CodexCliCaps` over `codex.cliCaps()`, registered by **both** shells.
- Every emitter threads it as `ApprovalCaps`: `approvalFlags` / `modeSupported` / `withPermissionMode` / `unsupportedModesNote` all take an optional trailing `caps`, and the **omitted** form resolves to the BASELINE `['on-request','never']`, the two values every measured codex accepts. That default is the design: a call site that forgets to thread its probe result loses a mode, never a launch.
- On 0.149.0+ `modeSupported('codex','manual')` is **false** and the derived note says so. Measured before concluding it: `-a unless-trusted` is refused, `-c approval_policy=untrusted` fails config load, and `--approve-for-me` routes approvals through *automatic* review.
- **Remote is unknown, never guessed**: an SSH node runs the HOST's codex and there is no remote codex probe yet (claude has one, at connect), so `codexApprovalCaps(ssh)` and the mirror's SSH slice publish nothing and fall to the baseline. Same for a relay tab (the guest's machine): its stub answers unknown on purpose.
- Surfaces: **Desktop** and **Server Edition** both probe their own machine (the server's `registerCodexCliIpc` is REAL, unlike its deliberately-false `registerCodexIdentityIpc`). **Mobile** gets the vocabulary as `MirrorSettings.codexApprovalValues`; desktop and server publish it, the iOS reader is the follow-up.
- The app-server **usage tier** (`usage/codex-usage.ts`) carried the same dead `-a untrusted` and silently returned null on every current CLI; it is now `-a never` with **no probe**, because `never` is in both vocabularies (measured against 0.148.0, 0.151.0 and 0.154.0) and `-s read-only` is what actually guards the user's files.

## Settings, resolver and the launch funnel

- `settings.claudePermissionMode` (global, default **`auto`**, a behavior change for existing users, who previously got a prompt per action) is overridden per project by `project.defaultPermissionMode`, persisted to `.nodeterm/project.json` (so a `bypassPermissions` override travels to everyone who clones the repo; the tab menu warns).
- Modes are `manual | auto | acceptEdits | plan | bypassPermissions`, labelled once in `PERMISSION_MODE_LABELS` (from which `ALL_PERMISSION_MODES` is derived, so the dropdown and the validator cannot desync).
- `resolvePermissionMode(project, settings)` is the resolver; `renderer/state/permissionMode.ts` `activePermissionMode(agentId)` binds it to the live stores and applies the version gate below, for `agentId === 'claude'` only.
- **`withPermissionMode(cmd, agentId, mode)` is the single funnel through which every agent-node launch site appends the flag**: new node, cold-restore resume, Branch, handoff/transfer, explain-commit, add-agent, canvas-control open-agent and team spawn.
- **Where the flag lands is decided at the composed layer** (`createAgentNode`), not in `withPermissionMode`. With no `argvPromptSeparator` (claude) it goes LAST, keeping the historical command byte-identical. With one (grok's `--`) it must go BEFORE the separator, because `--` is end-of-options and a flag after it is a positional, silently swallowed into the prompt or a clap usage error. Assert that at `createAgentNode`: a `withPermissionMode` test passes while the composed line is wrong. Gemini and codex declare no separator, so their flag goes last and their command lines stay byte-identical; grok is the only agent taking the other branch.
- UI: Settings → Agents, and the tab ⌄ menu for the per-project override.
- The setting and the per-project override apply to **terminal (CLI) agent nodes only** (the SDK chat node, which never honored it, was removed 2026-07).

### Who chose the mode: a git-shared value never keys a local gate alone

`project.defaultPermissionMode` rides `project.json`, which is GIT-SHARED, and travels to everyone who clones the repo. Binding a confirmation-skip to "the mode is bypassPermissions" would let a cloned repository silently switch off a user's destructive-action gate. So ask `resolvePermissionModeWithSource` WHO chose the value (`project` / `global` / `default`) and act only on the user's own machine-local choice, and keep `default` distinct from `global`: reading an unset setting as a deliberate choice is reading consent into silence. Anything machine-local goes in `settings.json`; nothing that grants a capability goes in `project.json`.

A machine-local DEFAULT for a project capability (`agentMessagingDefault`) is allowed only because it answers ABSENCE: an explicit `true` in `project.json` still needs this machine's recorded answer, an explicit `false` still wins, and "off" must therefore be written as a literal `false`. Read grants through `projectCapabilityGrantedFor(project, cap, settings)`, never the file bit.

## Claude version gate (`auto` only)

`--permission-mode auto` exists only in **Claude Code 2.1.71 and later**; older CLIs validate the value against their own choices list and **exit 1**, and `auto` is the default, so an ungated flag would kill every Claude launch on an older CLI.

- The CLI is probed (`core/claude-cli.ts`, `claude --version`, memoized, registered on `CorePlatform` so **both** shells serve it; reached from the renderer via `window.nodeTerminal.claude.cliCaps()`, with a **real** ws-bridge implementation) and `gatePermissionMode(mode, autoSupported)` degrades **only `auto`**, and only to `manual` = **no flag** = the bare pre-feature command.
- Everything **fails open**: unknown or unreadable version, a probe that failed or has not answered yet gives the bare command, never a blocked launch. The other four modes are never touched by the gate, and the user's *setting* stays `auto` (only the emitted command line changes).
- **SSH projects** are gated on the **remote** host's CLI, never the local one. `SshProjectManager.connect` probes `claude --version` on the host (through a login shell, since an ssh exec channel's rc file usually bails out early, with `$HOME/.local/bin` + `$HOME/.claude/local` prepended to PATH: the official installer targets `~/.local/bin`, which a stock root `.profile` never adds, so a host whose interactive shells run claude fine still probed "not found" and silently degraded `auto` to manual) and caches the answer on the connection into `useSshConn`. Not connected or not yet probed means no `auto` flag.
- A FAILED remote probe (claude not found, often a transient login-shell hiccup) **retries on a bounded backoff** (`PROBE_RETRY_DELAYS_MS`; every attempt pushes its answer immediately so launch waiters never block on the retry tail; a definite version, old or new, never retries), and the status event carries `remoteClaudeVersion` (`null` = probe failed) beside the boolean.
- The cold-restore relaunch `await`s the (shell-warmed) local probe because it fires on mount, and on an SSH project whose resolved mode is `auto` it also waits (`SSH_AUTO_PROBE_WAIT_MS`, bounded, fail-open) for the REMOTE probe's first answer, which races the same mount.
- Because the degrade is silent by design, the tab menu's Auto rows surface it: `sshAutoModeHint` (tri-state `useSshConn.autoPermAnswer` + probed version) puts a warning mark and tooltip on "Auto" / "Use global (Auto)" for an SSH project whose remote CLI is too old, missing or not yet probed.
- **No other agent inherits this gate.** Grok has accepted every mode since 1.0.0 and gemini/codex accept theirs on the versions measured, so gating any of them on a `claude --version` probe would downgrade their sessions on a machine whose claude is old or absent. `activePermissionMode` gates only `'claude'`, `ensureActivePermissionMode` awaits the probes only for `'claude'`, and `sshAutoModeHint`'s copy names Claude in every sentence for the same reason. An agent needing its own gate adds one beside claude's.

## Security: re-validate at the interpolation site

Mode values come from hand-editable, git-shared JSON and end up interpolated into a shell command line (tmux `send-keys`), so `permissionModeFlag` **re-validates** the mode at the interpolation site (the type is compile-time only). An unrecognized mode yields **no flag**, i.e. the bare, safe command. `'manual'` likewise yields no flag, reproducing the pre-feature command bit-for-bit. `isPermissionMode` at the top of `approvalFlags` closes the forged-`constructor` lookup case.

## Codex NEEDS YOU under its auto-reviewer (#1171)

`core/agents/codex-permission-gate.ts`, in the hook server, so both shells get it. With `approvals_reviewer = "auto_review"` codex fires `PermissionRequest`, then its own reviewer subagent approves or refuses with nobody asked; no hook fires until the approved command's `PostToolUse`, so the node read NEEDS YOU for the review plus the whole command.

Measured (2026-10-16, private `CODEX_HOME` + fake Responses model, codex 0.156.1 `exec` and 0.163.0 TUI; fixture `shared/agents/__fixtures__/codex/auto-review-permission.json`): the hook payload has NO reviewer field; the rollout's per-turn `turn_context` carries `approvals_reviewer` and a `turn_id` equal to the hook's, already on disk when the hook runs; `guardian_subagent` is recorded as `auto_review`; a failed review rejects to the agent, never to a person.

Rules:

- A closed match (the `turn_context` for THIS `turn_id`, value in `CODEX_AUTO_REVIEWERS`) suppresses the event, so the node keeps PreToolUse's `working`.
- Everything else is published unchanged as `waiting`: no `turn_id`, unknown value, a path outside the codex sessions roots (system `$CODEX_HOME/sessions`, managed `~/.nodeterm/cx/<16hex>/sessions`), unreadable file (`O_NOFOLLOW`), no match within the 16 MB backward scan, a read over 500 ms.
- The hook reply never waits on the read; later events of the node wait behind it.
- A remote (SSH) codex node's rollout is on its host, so it behaves as before.
