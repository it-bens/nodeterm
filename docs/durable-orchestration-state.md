# Durable orchestration state

Four facts a canvas-control orchestration leans on (the delivery queue, station reports, hand-over holds and the `--request-id` ledger) survive an app or Server Edition restart. They are mirrored to `<userData>/orchestration-state/<kind>.json` through one storage module, `core/durable-state.ts`. See [agent messaging](agent-messaging.md) for the queue and [atomic writes](atomic-writes.md) for the rename rules these files follow.

## Why

These facts used to live only in process memory, so a restart erased them while the work they described kept going:

- a `send` answered `queued` vanished while its sender believed it would be delivered;
- every station report vanished, so each `--after-success` dependent read BLOCKED and needed the play button or `run`;
- the `--request-id` ledger emptied, so a retry after a restart re-ran an open that had already happened;
- the plain `--after` hand-over hold (#1052) was forgotten, so a dependent fired on the first `done` after a restart.

All four go through `DurableFactFile` plus a per-fact `DurableFactSpec`, so a new fact is one more spec, never another copy of the read / sanitize / write / flush code.

**To add a fact:** write a `DurableFactSpec` (kind, version, a sanitizer that DROPS what it cannot trust, a cap) rather than another store. Decide, and write in the fact's header, what a restart MEANS for it (a TTL that kept running, a session that may have changed). Load it at boot in BOTH shells after anything it reads (the status mirror). Test it by writing through one instance and reading through a new one. Never put it in `.nodeterm/project.json`: it is one machine's run state.

## Storage rules

- **Machine-local only.** userData, never `.nodeterm/project.json`: a queued body, a station verdict and a control reply are one machine's run state, and a clone must not inherit them. Written 0600 (a queued message's body is the user's text). Relay tabs and the phone: N/A, because the facts belong to the core that executes the calls; a relay tab talks to the host's core, whose files these are.
- **The file is hostile input.** Envelope `{kind, version, savedAt, records}`. An unknown kind or version starts empty (warned); a record the fact's sanitizer refuses is DROPPED, never repaired; lists are capped; a file that is not JSON, is too large (16 MB) or is not an envelope is set aside as `<file>.corrupt` (one copy) and the fact starts EMPTY with a warning. Loading never throws: a bad orchestration file must not take the boot with it.
- **Writes** are a unique `wx` temp then `renameAtomicSync` (the fs-atomic guard applies), coalesced over 50 ms, one in flight, latest snapshot wins. `flushAllDurableFactsSync()` runs on the desktop's second `before-quit` pass, `hookServer.stop()` flushes the ledger, and the Server Edition's canvas-control `stop()` disposes its three files (queue, reports, holds). A crash inside the 50 ms window loses that window.
- **A sync flush is never overwritten by an older async write.** Reproduced in review: `save([1])`, `flush()`, `save([2])`, `flushSync()` left `[1]` on disk. `flushSync` bumps a generation, and the async path checks it and renames in one synchronous step, dropping its temp when stale.
- **Every owner budgets its bytes under the 16 MB load limit**, because a file past it is set aside whole (reproduced: 80 queued 250K-char bodies wrote 20 MB and 0 of 80 came back). The queue writes full entries up to 8 MB of JSON and the rest REDUCED (no body, short fields only: `bodyOmitted`, which restore turns into an expiry the sender hears about). The ledger writes replies up to 8 MB and the rest as UNKNOWN rows (still refused, never re-run).
- **Only the instance that owns the hook endpoint owns the facts.** The ledger is loaded by `hookServer.start()`, which fails for a second instance. The queue, reports and holds follow the same rule (`standDown()` when `startForApp()` returned a warning: desktop `main/index.ts`, Server Edition `ownsDurableState`), so a second instance on the same userData neither expires messages the live one holds nor overwrites its files.
- **The desktop queue restore waits for the workspace index** (`restoreDeliveryQueue(…, {ready: workspaceStore.load({sideline:false})})`). An entry that lapsed during the downtime is expired there, and its sender leg resolves the sender's project and board log through the index, which nothing else has loaded at that point of boot (otherwise the expiry reached only the in-memory trace ring). The Server Edition already awaited the load before canvas control.
- **Boot order is load-bearing** (desktop `main/index.ts` right after `initAgentStatusMirror()`; Server Edition inside `initServerCanvasControl`, which runs after the mirror): station reports first (their session check reads the restored mirror), then the hand-over holds, then the delivery queue. Its restore replays a `queued` hand-over per waiting message (so "work pending" is REBUILT from the queue, not stored twice) and settles, without landing, every message that lapsed while the app was down, which withdraws the station's report exactly like an in-run expiry.
- **Measured** (Linux dev host, ledger at its 4096-row cap with realistic open replies): 377 bytes a row, a 1.5 MB file, 11.5 ms per full write, 16.6 ms to load at boot; a 20-row ledger writes in 0.41 ms. The queue is bounded far lower (16 per target), the reports at 1000.

## What a restart means, per fact

### Queued message (`core/agents/delivery-queue.ts`, `QUEUE_FACT`)

- **The TTL keeps running while the app is down** (deadline = `enqueuedAt + ttlMs`, wall clock). A message whose deadline passed is EXPIRED at restore: traced `expired`, the sender told through `onExpired`, never delivered late and never dropped in silence. The rest re-arm with the time they have left (a clock that went backwards cannot stretch one past a full TTL).
- **Same session only.** At enqueue the target's agent and session id are recorded from the status mirror (`bindingOf`). A restored entry flushes only when the target's current session and agent are the recorded ones (`restoredBindingVerdict`). A different one (respawn, `/clear`, another agent in the pane) ends it `targetGone` with nothing typed; an entry with no recorded session is refused the same way; a target that has not named a session yet WAITS (TTL running). In-run entries are unchanged.
- **The whole gate chain still runs at flush.** After an app restart where tmux survived, pane ownership is unproven (`pane-ownership.ts` records only on a fresh spawn), so the first flush is refused `notPermitted` and the sender is told. That is still strictly better than the silent loss it replaces. After a machine reboot the cold-restored pane is a fresh spawn, so a message for a session that resumed under its old id delivers. The Server Edition's creator ledger is process-local too, so a restored message there is refused `caller-not-owner`.
- **Never replayed into a pane:** a board comment (only the local user, typing in this app, may trigger one; a message read back off disk must not speak as a person) and an app-composed station notice. Both, and a body over 256 KB (written without it), are expired at restore so their row or sender still hears the end. Nothing is flushed at boot: the first flush waits for the target's next `done`.
- **Threat model:** the file is as trustworthy as the per-node token files beside it, since a same-user process can write either. A forged `send` entry still runs every gate at flush.
- **Delivery is at most once across a crash.** A flush writes the entry OFF disk before its attempt (claim before effect), so a crash mid-delivery loses that one message rather than typing it twice after the next boot. Lapsed entries at restore are never inserted into the live lists (so no flush can deliver one while the expiries are reported) and take no capacity slot.

### Station report (`core/station-outcome-store.ts`, `OUTCOME_FACT`)

Stored with the station's session and agent (`sessionOf`). At load, a report whose recorded session or agent differs from what the restored mirror now says for that node is dropped. Afterwards a `SessionStart` naming a different session or agent withdraws it, in-run too, since a report is about a task in one conversation (a child's session event, `subagentType`, never counts). An unknown side keeps it. The withdrawal runs in `emitAgentStatus` BEFORE the renderer hears the event, so no dependent can fire on the stale report in between.

### Hand-over hold (`core/station-handover.ts`, `HANDOVER_FACT`)

The plain `--after` fact from #1052. The HOLDING stations are stored (`handedAt`, the current turn's start and state, the background flag); `queued` is rebuilt from the queue's replay. Times are wall clock, so a turn that began before the restart but after the hand-over still ends the hold. Hook events are lost while the app is down, so a turn that ended during the downtime is not seen and the hold lasts until the next turn end (the holding direction; the play button or `run` releases it). Not bound to a session: a respawned station still owes the work it was handed. Loaded after the reports and before the queue.

### Request ledger (`core/control-request-ledger.ts`, `CONTROL_REQUEST_FACT`)

Owned by the hook server's route. Settled rows replay. A row in flight at shutdown (or a crash) comes back UNKNOWN: refused, never re-run, and nothing can settle it any more because its late answer died with the old process. A reply over 64 KB is written as UNKNOWN rather than dropped, since a missing row would let the retry run.
