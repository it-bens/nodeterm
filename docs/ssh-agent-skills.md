# nodeterm skills in SSH projects

How the agent-facing skills (`manage-nodeterm-canvas`, `get-linked-context`) reach an agent
that is running on a **remote host** in an SSH project.

## The problem

Both skills were installed only into the desktop's own home:

- `manage-nodeterm-canvas` → `~/.claude/skills/…` + `<userData>/canvas-control/`
- `get-linked-context` → `~/.claude/skills/…` + `<userData>/context-links/`

An SSH project's agent runs on the **host**, reads the **host's** `~/.claude/skills`, and finds
nothing. Copying the files over would not have helped either: both shims were generated as

```sh
ELECTRON_RUN_AS_NODE=1 exec "<the desktop's Electron binary>" "<…>/cli.mjs" "$@"
```

— a path that exists on exactly one machine. And the Node CLI they exec'd talked to the hook
server over `127.0.0.1:<port>`, which on the host is the host's own loopback, not the desktop's.

## The transport (already there)

Nothing new had to be invented: an SSH project already runs a **reverse unix-socket tunnel** from
the host back to the desktop's loopback hook server, because that is how remote agents report
status (`RemoteHooks.setup`, `docs/`-less but see `src/main/remote-ssh/remote-hooks.ts`):

```
remote agent → curl --unix-socket ~/.nodeterm/hook-<projectId>.sock
             → ssh -R → desktop 127.0.0.1:<hook port>
```

The per-project endpoint file `~/.nodeterm/hook-endpoint-<projectId>.env` carries the live
`NODETERM_HOOK_SOCK` / `_TOKEN` / `_VERSION`, and every remote session's tmux env already points
at it (`NODETERM_HOOK_ENDPOINT`) along with `NODETERM_NODE_ID`. So the design rule is:

> **The remote side is a thin client.** It ships no parsing, no state and no app knowledge — it
> POSTs over the existing tunnel and prints what comes back.

## What shipped: canvas control over SSH

1. **The CLI is now POSIX sh + curl** (`CONTROL_SHIM_SCRIPT` in `canvas-control-core.ts`),
   replacing the Electron-as-Node CLI. It carries **no machine-specific paths**, so the same
   script is installed on the desktop and on the host. It picks its transport from the endpoint
   it was given: `--unix-socket $NODETERM_HOOK_SOCK` when one is advertised (SSH), else
   `127.0.0.1:$NODETERM_HOOK_PORT` (desktop). `curl` is not a new dependency — the managed hook
   script already requires it.

   The request is **form-urlencoded**, not JSON: `curl --data-urlencode` is the only escaping sh
   can be trusted with. Values like `--prompt`, `--html` and `--team` routinely contain quotes,
   newlines and `$`, and hand-built JSON in sh breaks on all three. `parseControlBody`
   (`hook-server.ts`) reads both dialects; the desktop's in-process callers still send JSON.
   A `text/plain` `Accept` makes the server render the reply, since sh has no JSON parser.

2. **`RemoteHooks.installCanvasControl`** writes the shim + `SKILL.md` onto the host, and merges
   the same marker-delimited instruction block into the host's `~/.codex/AGENTS.md`,
   `~/.gemini/GEMINI.md`, copilot's `copilot-instructions.md` and
   `${XDG_CONFIG_HOME:-~/.config}/opencode/AGENTS.md` (that last path is expanded by the **remote**
   shell — the desktop's XDG value says nothing about the host). Managed Claude accounts, whose
   sessions resolve skills relative to `CLAUDE_CONFIG_DIR` and never see `~/.claude/skills`, get
   their own copy (`installCanvasSkillIntoAccountDir` when the account is added). On a connect
   these are not written blind: `RemoteHooks.refreshAgentTools` checks them (see
   [Keeping the host current](#keeping-the-host-current)) and rewrites only what differs.

3. **The env gap** — `remoteHookEnvArgs` injected only endpoint/node-id/version, so a remote
   session inherited neither `NODETERM_AGENT_ID` nor `NODETERM_CANVAS_CONTROL`. The CLI gates
   itself on the latter, so the skill would have been inert on every SSH node even once
   installed. It now mirrors the local `hookServer.buildPtyEnv` exactly, `canControlCanvas`
   gate included.

### Flag syntax, and why a verb must not depend on it

Flags are `--flag value` or `--flag=value`, and a flag may carry no value at all, anywhere on the
line. The `=` form is the **only** way to pass a value that itself starts with `--`
(`open-terminal --cmd=--version`); written as two tokens, a leading `--` is read as the next flag,
so `--text --oops` sends an empty `--text` plus a stray `--oops`.

That is a deliberate trade. The loop used to consume the token after any `--flag`
*unconditionally*, so `--read --node b1` became `arg.read=--node` with `b1` dropped, and a
valueless flag was expressible only as the last token on the line. Both failures were **silent** —
the request stayed well-formed and the server answered about the wrong flag, which sends the next
debugger to the verb table instead of to the shim.

**The staleness window this lives in:** the shim is rewritten locally at every app boot, and an
SSH host's copy is checked on every connect and brought up to this build (below), so a parser
improvement reaches remote agent nodes on the first connect after the update. A host can still run
an older loop for a while — its tunnel is down, the file is unreadable, or a second desktop on an
older build shares the host account — with nothing on the wire to say which loop is running. New
verbs are therefore designed to parse identically under both loops: give every flag a value and
the old and new loops agree.

### Keeping the host current

Everything above is generated by the app, so a host's copy is only as current as the last build
that wrote it. `RemoteHooks.refreshAgentTools` keeps it current, for the canvas and context shims,
both skills (system and every managed account pinned to the host), and our instruction blocks:

- **One probe, one round trip.** The host runs POSIX `cksum` over each file (over exactly our
  marker span for an instruction file), and the desktop compares that with the checksum of the
  bytes it would write. Only what differs is rewritten; a current host costs one read and no
  write. On a host with no `cksum`, the files it can read are written without comparison (as every
  connect did before) and the blocks merged; missing, unreadable and gated files are still told
  apart.
- **When.** Every connect (a fresh master, or the one that survived an app restart) and every
  reverse-tunnel repair. The 45 s watchdog costs nothing once the host is confirmed for this run,
  retries an unconfirmed host on a 1/5/15-minute backoff, and looks again hourly — the only thing
  that can change the files mid-run is someone else (another desktop, a hand edit).
- **Never.** A file that cannot be read is not written over; a managed account whose dir is gone
  from the host does not get it back (re-checked in the write itself); a probe reply that does not
  parse changes nothing. A stray end marker left above a hand-deleted block is skipped — by the
  merge and the probe alike — so it no longer makes every check append a new copy.
- **A running agent** keeps what it already read: the shim's `help` is current as soon as the
  file is, a Claude skill is re-read when it is next invoked, and codex / gemini / opencode read
  their instruction files at session start.

### Gating and failure behavior

Install is gated on **both** a resolved remote `$HOME` (every remote path must be absolute — a
literal `~` does not expand where these strings land) and a **verified** tunnel: `setup()` only
returns an endpoint path after proving the reverse forward reaches *this* app run. A skill
pointing at a dead socket is worse than no skill, because the agent retries instead of telling
the user canvas control is unavailable.

Everything else fails open, per step: an unwritable instruction file, a host without `curl`, a
dropped connection — the session runs, just without canvas control. Outside a nodeterm-spawned
session the shim exits immediately on its env gate, so it stays inert in the user's own
terminals (same discipline as the managed hook script).

## What shipped: context link over SSH

`get-linked-context` could not use the canvas-control trick as-is: its CLI was a ~230-line parser
for three transcript formats (claude JSONL, codex rollout, gemini event-sourced, plus an opencode
export), too much to express in sh — and the host may have no `node` at all.

So the same rule was applied one level deeper: **the desktop does the reading and the parsing.**

1. **The parsing moved out of the CLI** into `core/context-link-render.ts` — ordinary, tested TS
   instead of JavaScript embedded in a template literal. `renderContextLink` takes a link document
   and three injected fetchers and returns the text to print; it knows nothing about ssh, fs, or
   tmux.
2. **A `/context-link/<verb>` route** on the hook server answers with `text/plain`, and
   `context.sh` — the sh+curl twin of `nodeterm.sh` — is the only client. The local path uses the
   same route, so there is one implementation, not a local one and a remote one that drift.
3. **Remote reads were already reachable** from the desktop: a remote node's hook-fed
   `transcriptPath` *is* a remote path, read over the ControlMaster with `RemoteFile.readTail`
   (bounded — a long session's transcript is tens of MB and a `summary` shows 15 lines), and
   `PtyManager.captureSession` was already remote-aware, so `terminal` needed no new code. The
   three deps are injected by `src/main` (`initContextLink(ptyManager, deps)`); the Server Edition
   passes none and keeps its local-only behavior.
4. **A missing link in the hook path**: the remote branch of the raw hook listener never called
   `setNodeTranscript`, so a remote node had no known transcript at all. It does now — with the
   path already jailed by `isSafeRemoteTranscriptPath`, so a forged POST cannot aim a link read at
   an arbitrary remote file.
5. **`resolveLinkTranscript` refuses the local locators for a remote node.** They search *this*
   machine's disk and would have returned some unrelated local session's file — the linked agent
   would have read a stranger's conversation with nothing to indicate it. For a remote node the
   hook-fed path is the only trustworthy source; without it there is simply nothing to read yet.

### Authorization

The link document is chosen by the **requester's own node id**, and `pickLinkNode` will only
return a node listed in it. So a caller holding the hook token still cannot read a node it was
never linked to, and cannot read *through* an edge it does not itself hold (the link map is
directional). The token remains the outer boundary, exactly as for status hooks. The one value
that reaches a remote command line — an opencode session id — is shell-quoted at that site.

`docs/`-worthy consequence: at the time of this change `src/server` did not call
`initContextLink`, so context link was absent on the Server Edition. **It is wired now** —
`src/server/context-link.ts` calls `initContextLink(ptyManager, {})` and keeps the link map in step
with the canvases on disk. It passes **no remote deps**, deliberately: that shell runs ON the host
whose transcripts and tmux it reads, so local-only is the complete answer there and SSH projects are
a desktop-only concept.

## Testing

`canvas-control-shim.test.ts` runs the **real shim** under `/bin/sh` against a **real** hook
server, over both a TCP port and a **unix socket** — the shim is generated source no compiler
checks, and it is now the only canvas-control client. It covers the nasty-value round trip
(quotes / newlines / `$` / backticks / backslashes), the positional forms, error propagation,
the env gate, a bad token, a dead socket, and reading coordinates from the endpoint file.

`context-link.cli.test.ts` does the same for the context-link shim, carrying over the fixtures and
expectations of the suite that tested the retired Node CLI, so the rewrite answers for the
behavior it inherited. `context-link.handler.test.ts` drives the real handler: authorization,
the local/remote read split, and the locator guard.

## Out of scope

- **Server Edition** — canvas control is not wired there at all (no `agent:control` handler),
  which this change does not alter. The shim is now portable enough to be reused when it is.
- **Mobile companion** — no canvas, so no canvas control to surface.
- **Relay remote nodes** — a different transport with no client fs; unaffected.

## Freshness check: details and rules

Additional facts about `RemoteHooks.refreshAgentTools` (`main/remote-ssh/agent-tools-freshness.ts`) beyond "Keeping the host current" above.

- **Scope since #744.** The plan holds only what the host's consent installs: shims and skills, no instruction blocks. An unanswered host is skipped.
- **Why it exists.** The canvas and context shims, both `SKILL.md` files and our blocks in the codex, gemini, copilot and opencode instruction files used to be written only by the establish path, blind, and never looked at again, so a host could keep another build's text for a whole run. A fire-and-forget install that failed open was never retried. A tunnel that failed verification at connect and was repaired later on the reuse branch (see [hook model](hook-model.md)) never got them at all. A managed account's skill was written once, when the account was added, so after every update each remote account session read the verb docs of the build that created the account. The obvious suspect is not one: an app update never lands on the reuse branch. `conns` is in memory, so the first connect after a relaunch adopts the ControlPersist orphan on the establish path, which always wrote; `ssh-project.test.ts` pins that it checks there too.
- **The stamp is the bytes.** One generated probe (one round trip, a few hundred bytes back) runs POSIX `cksum` over every file the host holds and, for an instruction file, over exactly the span `merge*Block` would replace (awk under `LC_ALL=C`: the first start marker through the first end marker, only when the end follows the start). That is compared with `posixCksum` of the exact bytes this build would write (`core/remote-ssh/posix-cksum.ts`, pinned against the real binary), and only what differs is rewritten, through the same appliers as the install. A current host costs the probe and no write. Nothing is embedded in the artifacts: a stamp line would be noise in every agent's context, would need a migration for hosts written by older builds, and would trust a file's claim about itself. `cksum` is the one checksum POSIX requires; CRC-32 plus length is not collision resistant and does not need to be, because this detects drift and is not a security check.
- **Hosts without `cksum`.** Ubuntu's own BusyBox build omits it, so such a host exists: missing, unreadable and gated files are still told apart there, the files it can read are written without comparison (what every connect did before) and the blocks merged. An awk that fails on a block is reported (`X`, through fd 3, because `awk | cksum` exits with cksum's status) and that block is merged, which writes only on a change. Only files actually written are logged as "rewrote" or make the outcome `refreshed`.
- **Test coverage.** The permanent suite runs the probe under every shell and awk the machine has (what the CI image has); a one-off manual run added BusyBox sh, zsh 5.9 and the one-true awk 20231127 (`NT_PROBE_EXTRA_AWK` / `_SH`). macOS's own awk (20200816) and BSD `cksum` have never been run; that is a Mac device check.
- **The end marker is searched after the start marker**, in both `merge*Block` functions and in the probe's awk alike. Taking the first end marker anywhere read a hand-deleted block's leftover end line as "no block": the merge appended a fresh copy every time, and with an hourly check a host's `AGENTS.md` grew by one block an hour (measured in review: 41,693, 81,279, 120,865, 160,451 bytes). The same merges run at boot for the desktop's and the Server Edition's own local instruction files (`initCanvasControl`, `initContextLink`), which grew by one block per launch.
- **Refusals.** A file that is not a readable regular file (a directory, a dangling dotfile link, no permission) is never written over, and the host is not called confirmed. A managed account's skill is refreshed only when its dir already exists, checked by the probe and again on the host in the write itself (`remoteAtomicWrite`'s `requireDir`), so a dir removed in between is not brought back by the parent `mkdir -p`. A report that does not parse changes nothing. Account ids from settings are re-validated (`isSafeAccountId`) before they become paths. The copilot block is judged at the host's `$COPILOT_HOME` only when the installer's validator would accept that value.
- **Cadence.** A connect (establish, including the post-relaunch orphan adoption) and a tunnel repair always check. The 45 s reuse branch costs nothing once this run has confirmed the host for the current expected set (content plus account list). An unconfirmed host is retried there on the tunnel-repair backoff (1/5/15 min). A confirmed host is looked at again hourly (`AGENT_TOOLS_RECHECK_MS`), because within a run only a writer outside it (another desktop, possibly an older build, on the same host account; a hand edit) can change the files. One check per host at a time: projects sharing a host share it.
- **A new agent-facing doc on a host goes into the artifact plan in `remote-hooks.ts`** (`canvasControlArtifacts`, `contextLinkArtifacts`, `accountSkillArtifacts`): shims, skills, instruction blocks. The installers and the probe both read it, so a file added there is written and kept current. A shim, skill or block written from anywhere else is written once and never checked again, so hosts keep the old text across app updates. This is only for those docs: the hook scripts and the agents' hook config belong to `setup()`'s ordered chain (after the verified tunnel and the endpoint file), and the endpoint file and node tokens carry credentials. None of those may be rewritten on a freshness cadence.
- **What a running agent sees.** The shim's `help` is answered by the shim itself (baked from the verb registry), so it is current the moment the file is. Claude reads a `SKILL.md` body when the skill is invoked; codex, gemini and opencode read their instruction files at session start, so a session started before a rewrite keeps the old text until it restarts. Nothing is typed into a pane to announce it.
- **Surfaces.** Desktop only (SSH projects are a desktop concept). The Server Edition runs on its host and rewrites its local shims at every boot. Mobile: N/A.
