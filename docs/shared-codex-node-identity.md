# Shared Codex node identity — the account-scoped ownership spine

A Codex canvas node talks to **one shared `codex app-server`** and owns **one thread** inside it.
The node↔thread mapping is what survives resumes, in-pane restarts, and app restarts. S6 adds the
**account** dimension: the same machine can hold the system login (`~/.codex`) alongside several
managed logins, and a thread's ownership record must name **which account** it belongs to, not just
which node — so one account can never be made to speak for another's threads.

This document covers the whole S6 feature — the account × machine model, the on-disk layout, the
switch and cross-machine protocols, the fail-closed properties, and the same-filesystem (U3) /
discovery (U1) constraints the probes settled. S6 shipped across nine PRs (the account model, the
identity proxy documented here, the rollout primitive, the relay daemon, local accounts + the switch,
SSH remote accounts, per-account usage, the Settings UI, and the per-node account picker); this
feature is **fully operable** — a Codex node carries an account scope, the picker stamps it, and the
switch/transfer verbs run. Every layer is tested against real primitives (real `/bin/sh`, real fs,
real HMAC, the real main-side switch handler); the composition is walked once end-to-end in
`src/main/codex-accounts-e2e.test.ts` (Task 9.2), and the human/device verifications that headless CI
cannot run are enumerated in [the acceptance gate](codex-accounts-acceptance.md).

Based on @Corvin's S6 design in external PR #112.

## The account × machine model

An account is **system** or **managed**, and it lives on **one machine**:

- **System account** — the login at `$CODEX_HOME` (or `~/.codex`). No id. Always resolves. A machine
  with no managed accounts behaves exactly as it did before S6 (Constraint 12).
- **Managed account** — a second (or third…) logged-in Codex identity, addressed by a validated id.
  Its home is private (`0700`) and isolated; only NON-secret installation assets (`config.toml`,
  `AGENTS.md`, `skills`, …) are symlinked in from the system home — never `auth.json`, never the
  thread DB, so a managed account acts only as **its own** login.
- **Remote account** — the same, but its home lives on an SSH host (`host` set). The desktop drives
  its lifecycle over SSH; the host runs the relay + import, not its own account-management IPC.

`id` empty/undefined at any call site means the system account. Ids arrive from hand-editable,
git-shared `settings.json` / `project.json`, so every id passes `isSafeAccountId`
(`^[A-Za-z0-9][A-Za-z0-9._-]*$`, must start alphanumeric) **before** it becomes a path component.

## On-disk home layout

| What | Where |
| --- | --- |
| System home | `$CODEX_HOME` or `~/.codex` |
| Managed home (local) | `~/.nodeterm/cx/<sha256(userDataDir ␀ accountId)[0..16]>`, mode `0700` |
| Managed home (remote) | `<remoteHome>/.nodeterm/cx/<sha256(accountId)[0..16]>` |
| App-server control socket | `<home>/app-server-control/app-server-control.sock` |
| Ownership records | `<userDataDir>/codex-thread-nodes/` (system: bare root; managed: `<accountId>/`) |

The digest is deliberately **short**: the app-server control Unix socket lives two levels below the
home and must stay under macOS `SUN_LEN`, which an Electron userData path plus a UUID already
overshoots. `userDataDir` is folded into the LOCAL digest so separate NodeTerm profiles never
collide; a remote host has one home root, so the remote digest is over `accountId` only. A
pre-migration long home (`<userData>/codex-accounts/<id>`) is moved to its short home at boot,
fail-closed (no-op if absent, if the target exists, or on an invalid id).

## Shared-daemon restart continuity

The control socket is not one process per canvas node. Every `codex --remote unix://` TUI for an
account scope rides one persistent app-server, so an app-server upgrade, repair, or crash severs all
of them at once. Their tmux panes and Codex rollouts remain intact, but without a client supervisor
each pane drops back to its shell and looks like a reset.

The generated launcher therefore owns the remote client for the lifetime of the bound thread:

1. It records the app-server control socket's inode before launching the TUI.
2. A normal exit or terminal signal returns unchanged.
3. After any other exit, it retries only when `codex app-server daemon version` does not report
   `status: running`, or when a known pre-launch socket inode changed. A failed inode read alone is
   unknown, not proof of a reset; an unrelated Codex failure against the same healthy generation
   returns its original status.
4. It starts a missing daemon through the same scrubbed environment used at preflight, then resumes
   the exact already-bound thread. Only the first launch receives the caller's prompt/options;
   recovery never replays them because that could submit the same user turn twice.
5. Three rapid resets stop the loop and print the exact manual `codex resume <thread>` command.

Protocol health is checked before lifecycle start. Codex's PID ownership record can become stale
while the socket remains responsive; that bookkeeping failure must not trigger a kill of live,
shared infrastructure. The behavior is failure-injection tested under real `/bin/sh` in
`src/core/codex-launcher-sh.test.ts`, including a mutation guard proving that a healthy
same-generation client error is not relaunched.

Desktop and Server Edition both wire this same core spine. The headless shell arms the
restart-stable record secret, registers thread start/bind handlers against its persisted canvas,
refreshes the launcher capability, and broadcasts identity mode through its normal browser RPC.
Mobile needs no separate implementation: it attaches to the Server Edition's existing tmux
session. Managed Codex account administration remains desktop-only; that is separate from
supervising the system account's shared app-server on the server host.

## The signing secret (both shells — Decision 1)

Every ownership record is HMAC-signed by the **one** restart-stable 32-byte node-auth secret
(`src/core/agents/node-auth-secret.ts`, `loadOrCreateNodeAuthSecret`). That secret already arms on
**both** shells and this proxy reuses it as-is:

- **Desktop:** sealed at rest via `safeStorage` → `node-auth-key.json` (ciphertext only, mode
  `0600`; no raw-secret fallback ever written).
- **Server Edition (no keychain):** 32 raw bytes at `node-auth-key.bin`, mode `0600`.

It is **confidential and fail-closed**: a malformed/wrong-length persisted state **throws** rather
than minting a fresh secret — rotating the key would orphan every bound thread on the machine — and
the single-flight cache clears on rejection so a healed machine retries. It is wired at boot in both
`src/main/index.ts` and `src/server/index.ts` via `setCodexThreadIdentityAuthSecret(...)`. Arming
the secret is not the same as writing records: only the desktop registers the handlers that write
any — see the resolver section below.

> No `safeStorage`-only secret module is introduced. @Corvin's `codex-node-auth-secret.ts` in
> PR #112 is `safeStorage`-ciphertext-only and throws on a headless server; the merged both-shells
> channel above is the correct mirror, and duplicating it as a keychain-only file would regress the
> Server Edition. This is the ratified Decision 1.

## The ownership records

`~/.nodeterm/codex-thread-nodes/` (under `CorePlatform.userDataDir`, **not** `~`):

- **System account** → the bare-root file `<root>/<threadId>` — the exact S4 layout, so a machine
  with no managed accounts is byte-for-byte unchanged and its legacy records keep resolving.
- **Managed account** → `<root>/<accountId>/<threadId>`.

Each record is a flat `key=value` text file, mode `0600`, **parsed as data, never sourced**:

```
accountId=<id or empty for system>
nodeId=<owning canvas node>
endpoint=<hook endpoint file path>
agentId=<the node's own agent id>
canvasControl=<1 or 0>
signature=<base64url HMAC>
```

### Why the record names the agent

`agentId` and `canvasControl` were once **constants in the sh prelude** (`codex`, granted). Both are
facts about the PANE that only `hookServer.buildPtyEnv` knows: it labels a node with the node's own
agent id — which for a custom agent declaring `baseAgent: 'codex'` is `custom:<uuid>`, **not**
`codex` — and gates the grant on `canControlCanvas`. A constant is the prelude asserting what it
cannot know. It mislabelled every custom codex-based node, and it asserted a grant that agrees with
the pane today only because `SHARED_IDENTITY_CAPABLE ⊆ CANVAS_CONTROL_CAPABLE` — a coincidence that
list's own comment invites the next shared-identity agent to break, at which point the prelude would
be handing a tool shell a capability its pane was denied.

Who decides what: the **pane echoes its own label** on `/codex-thread/{start,bind}` (a tmux session
outlives the app, so after a restart nothing server-side still remembers what agent a node runs, and
the pane's environment is the only durable holder). The **grant is never echoed** — the route derives
it with `canControlCanvas`, the same predicate in the same process that gated the pane, so there is
exactly one decider and a forged agent id cannot manufacture a grant the capability table refuses.
Echoing the label is not a capability claim in any case: reaching that route requires a token this
instance minted for that node, so the caller *is* the node, and the record it shapes is re-exported
only into that node's own tool shells.

### The signature binds the full 6-tuple

```
signature = base64url( HMAC-SHA256(
  key, threadId ␀ accountScope ␀ nodeId ␀ hookEndpoint ␀ agentId ␀ canvasControl ) )
```

`accountScope` is the account id, or the literal `system` for the system account (empty id
normalised). Because the scope is inside the preimage, a record signed for account **A** verifies
only under scope **A** — copying it byte-for-byte into account **B**'s directory fails the MAC.
`canvasControl` is inside it for the same class of reason: it names a capability the prelude exports
into an agent's shell, so an unsigned copy would be a grant anyone able to write the file could add.
Verification is `timingSafeEqual`.

**The three preimage generations are SELECTED by the record's shape, never tried in turn.** An
`agentId=` line ⇒ the 6-tuple and only that; no agent line but an `accountId=` line ⇒ the 4-tuple
(an S6-era record); neither, at the system scope ⇒ the original 3-tuple. Falling through would be
the door by which an agent id could be stripped or rewritten and the record still accepted. A
pre-agent record is read with the implied values **`codex` + granted** — exactly what it meant when
it was written, since it came from this same spine and codex is unconditionally canvas-control-capable
— and that fallback is keyed on the **line being absent**, never on the value being empty, so nothing
that names an agent can land back on the guess. `bindCodexThreadIdentity` likewise never downgrades a
record that already names its agent, and never promotes a pre-agent record's *implication* into a
signed claim.

## Fail-closed ambiguity (the house rule, reused)

Ownership resolution reuses the merged fail-closed posture (`pane-ownership.ts`,
`node-identity-policy.ts`): **an owner that cannot be proven is denied.**

- `resolveCodexThreadNodeIdentity(threadId)` with no account hint (the shared tool shell that knows
  only a bare thread id) scans **every** scope and returns an owner **only when `owners.size === 1`**.
  The same thread id owned by two different accounts resolves to **nothing**.
- `resolveCodexThreadNodeIdentity(threadId, root, accountId)` with an explicit account resolves
  within that one scope.
- `codexThreadIdentityHasLiveConflict(...)` reports a conflict only when **two different live
  nodes** own the thread across scopes.

## The POSIX-sh resolver

`codexThreadIdentityResolverSh(root)` is prepended to every managed hook script and to both LOCAL
agent shims (`nodeterm.sh` and `context.sh`) before their early environment gates. A
shared-app-server tool shell inherits `CODEX_THREAD_ID` but not the pane's `NODETERM_*` (see probe
U5, `docs/superpowers/probes/2026-08-codex-tool-shell-env.md`), so this prelude recovers the binding.
Without it, Codex hook status worked while canvas control and linked-context reads misdiagnosed the
same first-class node as being outside nodeterm. The machine-neutral shim constants copied to SSH
hosts deliberately omit the local record path; each local installer builds its own copy with its
own `codexThreadIdentityRoot()`.

**The shim half is shared; the RECORD half is desktop-only until the Server Edition registers the
handlers.** Both statements matter, and reading only the first overstates what that edition has.
The prelude reaches it byte-for-byte — `context-link.ts`'s `writeCliFiles` is core, and both shells
arm the signing secret (above) — but `src/server/handlers/index.ts` deliberately registers no
`setCodexThreadStartHandler` / `setCodexThreadBindHandler`, and `src/main/index.ts` is the only
non-test caller of `writeCodexThreadIdentity` / `bindCodexThreadIdentity`. **No ownership record is
ever written on that shell**, so the resolver always finds none and takes its fallback: it exports
nothing, and the shims stay gated exactly as they were before this prelude existed. That is
coherent rather than merely missing — the same file answers `UNKNOWN_CODEX_IDENTITY_CAPS`
(`shared: false`), so a Codex node there launches the bare `codex` with its own app-server, and a
tool shell whose identity would need recovering does not exist. It becomes a real gap the day the
Server Edition grows the shared app-server, and what closes it is those two registrations — not a
second copy of the prelude.

The resolver:

- reads `NODETERM_CODEX_ACCOUNT_ID` from the daemon env to pick the scope;
- a known safe account id ⇒ reads **only** that account's record, and requires the record's
  `accountId=` line to **agree with the daemon scope**;
- an **empty** account id ⇒ scans every scope (bare-root system + each managed subdir) and binds
  **only when exactly one candidate matches** (`nt_codex_matches -eq 1`) — two accounts holding the
  same thread id change nothing;
- it cannot verify the HMAC (no key in an agent's shell), so it re-validates every recovered field's
  charset and the account-line/scope agreement before exporting. Its behaviour is proven under real
  `/bin/sh` against a real on-disk scope tree in `codex-thread-identity-sh.test.ts`.

**It exports what the record says; it does not decide.** `NODETERM_AGENT_ID` comes from the record's
`agentId`, and `NODETERM_CANVAS_CONTROL=1` is exported **only** when the record's `canvasControl` says
so — left UNSET otherwise, absent rather than `0`, the same shape `buildPtyEnv` produces and the shape
both shims' `[ -z … ]` gates expect. Withholding is the honest degrade: a tool shell loses a verb its
pane still has, whereas asserting a grant the pane was denied is the widening direction. A pre-agent
record (no `agentId=` line) means `codex` with the grant; a record whose agent id is outside the
accepted alphabet resolves **nothing at all** rather than falling back, because a record we did not
write is not one to trust field by field.

## The `NODETERM_*` env gate was never a security boundary

Worth stating plainly, because both the prelude and the two shims *look* like access control: they
open with `[ -z "$NODETERM_NODE_ID" ]` / `[ -z "$NODETERM_CANVAS_CONTROL" ]` and refuse to do
anything without them. Widening what SETS those variables — which is what recovering a shared-Codex
tool shell's identity does — therefore reads like widening what a local attacker can reach. It is
not, and this section is the argument.

Anyone who can run `nodeterm.sh` at all is already running as the user, in a shell, and can simply
`export NODETERM_NODE_ID=… NODETERM_CANVAS_CONTROL=1` by hand. The variables are a **routing and
no-op gate** — they tell a generated script which node it belongs to and keep it inert in the user's
ordinary terminals — not an authorization check. Nothing server-side consults them; they never
travel to the hook server at all.

What actually authorizes is on the other side of the socket:

- **Every route requires the app-wide bearer** (`X-NodeTerm-Hook-Token`, checked before any path
  dispatch in `hook-server.ts`), which lives in the 0600 endpoint file.
- **`/control/*` and `/context-link/*` additionally require the per-node capability**
  (`X-NodeTerm-Node-Token`) through the single `identityGate` → `verifyNodeToken` →
  `controlPolicy` path. The `nodeId` a caller names is a body field, and it is the token — not the
  environment — that decides whether that caller may speak for it. `forged` is a bare 403, and the
  strict verbs demand a `verified` verdict outright.
- **Context link narrows again by construction**: the link document is selected by the REQUESTER's
  node id (`linkDocs.get(req.nodeId)`), so a token-holding caller can only read the nodes in its own
  directional link map — an env var cannot widen that set.
- **`/codex-thread/{start,bind}`** — the routes that shape these very records — are strict: only a
  `verified` per-node token proceeds.

So the identity prelude changes *which sessions are correctly attributed to their node*, not *what
anyone is permitted to do*. This PR does not change the threat model. The one thing that genuinely
would is a value the prelude exports that nothing re-derives — which is exactly why
`canvasControl` is signed and why the route computes it with `canControlCanvas` rather than
accepting the pane's word for it.

## Supply-chain guard

Account ids arrive from hand-editable `settings.json` / `project.json`. Every id passes
`ACCOUNT_ID_RE` (`^[A-Za-z0-9][A-Za-z0-9._-]*$`, must start alphanumeric) — via `accountScope()` in
the proxy, the launcher's own sh check, and the hook-server route — **before** it becomes a
directory scope, so `.`/`..`/leading-separator/`a/b`/absolute ids are refused at the door. The
launcher threads the account id through the `/codex-thread/{start,bind}` POST **body**, never on
argv (Constraint 6); a bad id falls back to plain `codex` rather than binding under a hostile scope,
and the server refuses it at `400` before the start handler can create an orphan thread.

## Spawn scope resolution (fail-closed)

`resolveCodexSessionScope(userDataDir, accountId, homeExists)` decides the `CODEX_HOME` +
`NODETERM_CODEX_ACCOUNT_ID` a Codex spawn runs under:

- **No id** → the system home. Always resolves, and it is written **explicitly** so it overwrites any
  managed scope inherited from a parent (tmux shares one server env) rather than silently acting as
  the wrong login.
- **An explicitly selected managed account whose home is missing** → **refuses**
  (`{ unavailable: 'codex-account' }`). It never falls back to the system home. This is deliberately
  **stricter** than the Claude sibling's first-spawn fallback: silently acting as the wrong login is a
  worse failure for an explicit switch than a refused spawn. The `pty-manager` maps `unavailable`
  straight through to the renderer.

The OAuth-shadowing env vars (`OPENAI_API_KEY`, `CODEX_API_KEY` — `AUTH_ENV_STRIP`) are removed from
the session env at spawn so a managed account always acts as its own `auth.json` login. That strip
touches only the **env**; no credential is ever placed on an argv (Constraint 6).

## The same-machine switch (three-phase, owner-authorized)

Moving a running node's conversation to another account **resumes the same conversation id, never
forks it**. The switch is a three-phase, TTL-bounded, owner-authorized protocol driven by the
renderer (`src/main/codex-accounts.ts`):

1. **plan** (`switch-thread`) — reserve both account ids (so a concurrent removal is blocked —
   Property 10), read the source rollout path off its app-server, and `planCodexRolloutExposure`
   (validate only, mutate nothing). Returns a `rollbackToken`.
2. **commit** (`commit-switch`) — `commitCodexRolloutExposure`: **atomically hardlink** the source
   rollout inode into the target account's `sessions/<same relative path>`. Same inode ⇒
   byte-identical conversation id. Only the **owning WebContents** may commit.
3. **finish** (`finish-switch`) — release the reservation. Refuses until commit actually ran, so a
   premature finish can never make the renderer recycle a node onto a target that has no rollout.

`rollback-switch`, the reservation TTL, and the owner being `destroyed` all release a reservation
without committing. The exposure is an **atomic, never-overwrite hardlink**: `link(2)` is
no-overwrite, an `EEXIST` collision is tolerated **only** when the existing target is already the same
inode (an idempotent re-expose), and the copy refuses to write **through** a symlinked directory
segment. The source `dev`/`ino` is re-verified before and after the link, so a mid-flight source swap
fails closed.

## The cross-machine transfer (local → SSH)

`transfer-thread-to-ssh` moves an **idle** local conversation to a remote account. The SOURCE leg
reuses `planCodexRolloutExposure`'s guards (regular file under `<home>/sessions/`, basename
`<threadId>.jsonl`, no symlinked segment) and then hands the upload + atomic remote install to the
SSH importer. Properties: the conversation id survives; the far side must **discover** the hardlinked
rollout before the node is recycled (verify-then-recycle — a false discovery is a **refused copy +
rollback**, never a silent one); the **local copy remains** (it is a hardlink, never moved); and an
existing remote rollout is **never overwritten** (`link` / `mv` fail closed on `EEXIST` / `exit 17`).
An absent importer (no live SSH manager) **fails closed** with a named error, never a silent success.

## Per-account usage (no mixing)

Usage is one **system row** plus one row **per managed account**, each keyed by its own `accountId`
and built independently — there is no reduce/merge step, so one account's numbers can never be
attributed to another. Every `fetchCodexUsage` return path (including `unavailable`) is stamped with
the account's identity, so an empty or failed read fails closed to **that** account, never to a
fabricated one; the system row stays explicitly un-owned (`accountId: undefined`). A throwing account
source yields **system-only**, never a made-up account. A cache fingerprint over the account set
busts stale numbers when an account is added, removed, or relabelled.

## Fail-closed properties (the house rules)

| Situation | Verdict |
| --- | --- |
| Explicitly selected account, home missing | Refuse the spawn — no system fallback |
| Same thread id owned by two account scopes, no hint | No owner (proxy, sh resolver, relay catalog all fail closed) |
| Rollout collision at target, different inode | Never overwrite (throws / `exit 17`) |
| Cross-mount rollout link | Named `EXDEV` error — no silent byte-copy fallback (U3) |
| Far side cannot discover the imported rollout | Roll back the import |
| Record signed for account A, filed under B | MAC fails — rejected |
| No signing secret armed | Record write throws; nothing unsigned is written |
| Usage source throws | System-only; never a fabricated account |
| Account id from `settings.json` that could escape the root | Refused at the door by `isSafeAccountId` |

## U1 / U3 — the two constraints the probes settled

- **U3 — same filesystem.** A same-machine switch links a rollout inode between two LOCAL homes, both
  under `$HOME` (`~/.codex` and `~/.nodeterm/cx`), measured same-device on the build host (ext4).
  `link(2)` throws `EXDEV` across mounts, and a `$HOME` subtree **can** be bind-mounted onto another
  volume, so v1 does **not** silently fall back to a byte copy — `commit` surfaces a named `EXDEV`
  error. A cross-mount copy fallback is explicitly out of scope for S6 v1.
- **U1 — live discovery.** The cross-machine copy rests on a running `app-server` discovering a
  hardlinked rollout **without a reindex**. Mitigated in code by verify-then-recycle (a false U1 is a
  refused copy + rollback, never silent), but the live-daemon confirmation is a **device
  verification**, owed on a real host — see the acceptance gate.

## Daemon isolation and launch capability

### Every nodeterm Codex TUI uses `--no-daemon` when supported

From codex-cli 0.157.0, `daemon_auto_start` is `stable, true` (0.156.1: experimental false; 0.148.0: no such feature). A plain `codex` TUI then starts or JOINS one background `app-server` per `CODEX_HOME`. That daemon keeps the environment of the pane that started it and outlives it. The daemon spawns tool shells and hook processes, while nodeterm identifies a node by `buildPtyEnv` environment.

Measured on 0.159.2 (private `CODEX_HOME`, private tmux socket, `env -i`): pane A (`NODETERM_NODE_ID=node-A`) started the daemon; pane B (`node-B`) then had a tool shell print `node-A` and each hook process logged `node-A`. Pane B's status, canvas-control verbs and context-link reads were attributed to pane A. `--no-daemon` returned pane C to `node-C` in both places. `-c features.daemon_auto_start=false` did NOT, because it still joins a running daemon; there is no environment switch. Transcript and three help pages: `src/core/__fixtures__/codex-daemon/`.

- **Feature-detected, fail open.** `core/codex-cli.ts` `codexNoDaemonFrom` reads the flag from option-header lines of the memoized `codex --help` used for approval vocabulary. `CodexCliCaps.noDaemon` travels in `ApprovalCaps` (`codexNoDaemon`), and `withCodexNoDaemon` (`shared/agents/codex-daemon.ts`) appends it in BOTH assemblers: fresh launch and resume (cold restore, restart, restart-with-model, account switch, transfer, headless Server opens, custom agents whose `baseAgent` is codex, launch-command override). Only literal `true` emits it: clap exits on an unknown option, so unprobed, remote-unknown or older CLIs receive the old line.
- **Await the answer before building a line.** After reboot every Codex node cold-restores in one tick. A line built before the probe lands starts the shared daemon with that node's env and all other nodes join it. `TerminalNode`'s four Codex sites (cold restore, fresh fallback, restart, wake) await `ensureCodexLaunchCaps(...)` (`renderer/state/codexCli.ts`, 3 s bound, fail open to the old line). Local waits for local probe; SSH waits for that host's `useSshConn` answer; non-Codex agents and relay tabs never wait. `nodes/codex-launch-caps-wiring.test.ts` pins this source-level wiring. `createAgentNode` is synchronous and reads only the landed answer, so a node created in roughly the first second after boot can miss it.
- **Relay tabs never use this machine's answer.** Their pane runs the host's Codex. `codexApprovalCaps(remote, projectId)` treats a relay-bound project (via `registerCodexRelayProjectCheck`) or `session.source === 'relay'` as remote-with-no-probe: no flag, baseline vocabulary.
- **One detection rule, two spellings.** `CODEX_NO_DAEMON_HELP_RE` / `_ERE` in `shared/agents/codex-daemon.ts` match an option header at indent at most 6 followed by whitespace or end of line, so `--no-daemon-x` is not the flag. TypeScript reader, launcher and remote probe all use it. A test runs the ERE through real `grep -E` beside the regex.
- **Never beside `--remote`.** Measured: `ERROR: --no-daemon cannot be used with --remote.` `buildCodexLauncherScript` strips it before `codex --remote unix:// resume`; plain-Codex fallback uses `nt_exec_plain`, which retains it, or adds it when its Codex advertises it (the SSH launcher's host was not probed here). A daemon a plain pane starts carries that pane's `NODETERM_NODE_ID`; the thread-identity prelude resolves only an empty one, so a plain launch can poison every managed thread. Our own start remains scrubbed `nt_start_app_server` (#350).
- **SSH probes the host binary.** `core/remote-ssh/codex-no-daemon-probe.ts` runs one marker-delimited `codex --help` through the login shell after connect, off the connect path, and publishes `{hostKey, supported}` on `connected` and reused-connect result. Renderer keeps answers per host (`useSshConn.codexNoDaemonByHost`). `codexProbeHostKey` is `user@host:port`, not `sshHostKey`: two containers behind one host at `root@localhost:2222` (0.159) and `:2223` (0.148) are different binaries. SSH mirror carries host true to the phone.
- `codex exec` (commit messages), `login`, `mcp` and `app-server` take no flag and are not TUI clients. Phone gets `MirrorSettings.codexNoDaemon`, local and per SSH slice (iOS reader follow-up). opencode was checked: no published release has `serve --service` (latest 1.18.33 and dev channel), and a plain TUI leaves no process behind.

Residuals: a `codex` typed by hand in a plain terminal, or through `open-terminal --cmd codex` / `write`, carries that node's `NODETERM_NODE_ID` and on 0.157+ can start the account daemon with it. Managed threads then keep leaked identity because the prelude does not replace a set id. Tool shells cannot prefer their record instead: a bind-refused fallback pane can legitimately run a thread another node's record names. Hooks differ: their payload names their own thread, so the hook script lets its record outrank inherited id unless `NODETERM_CODEX_PLAIN=1`. A launch-command override or custom `launchCmd` that executes a different Codex from PATH (for example `npx @openai/codex@0.148.0`) can receive PATH's flag and die; drop the pin or add the flag to the command. The Windows argv planner (`core/agent-launch.ts`) carries the flag but has no production caller; Windows-native Codex is unmeasured.

Machines that ran a pre-fix build keep mis-attribution until they recycle: already joined panes and the daemon's `pid-update-loop` keep the original daemon environment. Do not kill the daemon, because unsupervised plain clients attached to it would die. Recovery: restart each Codex node (node menu, Restart, or close and reopen), then from a shell without any `NODETERM_*` variables run `codex app-server daemon restart` once per account, with that account's `CODEX_HOME`.

Device checklist: (1) macOS desktop, npm Codex >= 0.157, two nodes each show its own RUNNING badge and `nodeterm list` row. (2) Standalone Codex with shared identity, a launcher-fallback node reports itself. (3) SSH host Codex >= 0.157, second remote node badge is its own after probe lands and flagless before. (4) Windows-native Codex: determine whether daemon exists; flag rides only when `--help` lists it. (5) Reboot a Mac with 5+ Codex nodes: cold restore gives each its own badge. (6) Upgraded machine: after recovery, `ps eww` on daemon contains no `NODETERM_NODE_ID`. (7) Two SSH projects on one host at different ports with different Codex versions: only newer lines carry flag.

### Shared daemon restarts do not restart terminal sessions

A shared daemon restart leaves tmux and Codex rollout/thread alive, but every `codex --remote unix://` TUI attached to that account's app-server socket exits. `buildCodexLauncherScript` therefore stays in the pane as a bounded transport supervisor after binding a thread instead of `exec`ing the remote TUI. On abnormal client exit it resumes that exact thread only if `app-server daemon version` no longer reports `status: running` or the known control-socket inode changed; a healthy generation returns the original status so deterministic CLI errors cannot relaunch forever. A reconnect never replays launch prompt/options, and three rapid resets stop with a manual `codex resume <thread>` receipt. Preflight probes live protocol health before lifecycle start: Codex's PID ownership record can be stale while the shared process remains responsive, and killing that "orphan" would fan one bookkeeping failure out across every node. Generated-shell tests run replaced-socket, missing-daemon, healthy-client-error and responsive-orphan cases under real `/bin/sh`; healthy-client-error is the mutation guard.

Electron and Server Edition wire the same signed record secret, thread start/bind handlers, capability refresh and UI identity events. Server composition is `server/codex-shared-identity.ts` and is behavior-tested. It used to answer `shared: false` (`UNKNOWN_CODEX_IDENTITY_CAPS`) deliberately, bypassing the launcher and leaving headless panes at their shells. It now wires the same spine after its secrets exist.

**No approval override rides remote resume** (issue #811). Measured on codex-cli 0.154.0 against a real thread on a running shared app-server under a pty: `codex --remote unix:// resume <thread> --ask-for-approval on-request` exits 1 with `Error: Permission overrides are not supported when resuming a remote task.` Removing the flag resumes and TUI stays up; `-c approval_policy=never` is refused identically. The rule is about OVERRIDE, not value/spelling. `nt_run_shared` strips `--ask-for-approval` / `-a`, both spaced and `=` forms, from first-launch argv.

The strip belongs in the LAUNCHER, not TypeScript that builds the line: identity setup failure above it ends in `exec codex "$@"`, plain Codex with no `--remote`, where 0.154 accepts the permission flag. Suppressing it above removes permission mode from exactly nodes unable to get managed identity. The `else` branch already resumes with no caller options, so first launch agrees with recovery. No capability gate exists: refusal is runtime, absent from `--help`, and fires after session lookup; probing needs the real resumable thread being launched.

## Hook identity recovery and record-controlled exports

A Codex HOOK under shared app-server is identified by its PAYLOAD, not environment (#1150, #1175). Measured on 0.162.0 with private `CODEX_HOME` and tmux socket: a hook process is the daemon child, carries daemon env and never `CODEX_THREAD_ID` (only tool shells get it), while stdin starts `{"session_id":"<thread>"}`. A scrubbed daemon therefore ran every managed node hook with no node id (silent exit), while a daemon a plain Codex pane started ran them under a long-gone node id.

The local managed script reads stdin FIRST, extracts leading `session_id` with builtins (anchored, so nested input never matches), then passes it to the prelude (`hookPayloadThread`). A record for that thread wins over inherited node id and stale endpoint variables are unset before the record endpoint file is sourced. Exception: a pane launcher ran in-process (`nt_exec_plain` exports `NODETERM_CODEX_PLAIN=1`, never on `--remote`) has its own environment, which stands. Tool shells and both shims keep old behavior (a set id is untouched). Event remains verified against bound node's 0600 token, so a forged payload buys no more than any same-user process could. Residuals: a bare Codex node with shared identity off whose thread has another node record reports as that node; a Codex typed inside a marked plain-pane tool shell starts a daemon carrying marker.

Every LOCAL generated sh client recovers shared-Codex identity before its environment gate. An account app-server tool shell carries `CODEX_THREAD_ID`, not pane `NODETERM_*`; managed hooks, local `nodeterm.sh` and local `context.sh` prepend `codexThreadIdentityResolverSh(codexThreadIdentityRoot())` before testing `NODETERM_NODE_ID` / `NODETERM_CANVAS_CONTROL`. SSH constants remain machine-neutral: local record root is invalid remotely and must never be baked into a remote copy. `main/remote-ssh/remote-shim-neutrality.guard.test.ts` enforces two legs: exported neutral bodies carry no record root/prelude, and `remote-hooks.ts` cannot name a parameterized builder. The prelude is shared; the record it reads is desktop-only. `src/main/index.ts` registers the writers; `wireServerCodexSharedIdentity` (`src/server/codex-shared-identity.ts`) does likewise at Server Edition boot, after secrets exist. The record's remote leg remains desktop-only.

The prelude EXPORTS what the record says, never decides it. `NODETERM_AGENT_ID` and `NODETERM_CANVAS_CONTROL` are `buildPtyEnv` answers about the pane. The node keeps its own agent label (for example `custom:<uuid>` for custom Codex harness), and `canControlCanvas` decides the grant. Constants once mislabelled custom nodes and asserted grants due to the coincidental `SHARED_IDENTITY_CAPABLE` subset of `CANVAS_CONTROL_CAPABLE`. The record now puts `agentId` and `canvasControl` inside its 6-tuple HMAC; prelude reads them and exports grant only when granted, otherwise UNSET (absent, never `0`, matching both shims' `[ -z … ]` gates). Pane echoes its own label at `/codex-thread/{start,bind}` because a tmux session outlives app state. The grant is never echoed: route derives with `canControlCanvas`, so forged id cannot manufacture a grant.

The three preimage generations are selected by record shape, never tried in turn. A record naming an agent must not verify under a preimage that ignores it. A pre-agent record's implied `codex` plus grant is keyed on LINE absence, not empty value, so an agent-bearing record cannot fall back to a guess. Environment variables were never a security boundary (a shim user can export them); the per-node token is.
