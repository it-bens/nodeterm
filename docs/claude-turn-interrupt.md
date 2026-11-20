# Interrupted Claude turns (Esc / Ctrl+C): the transcript marker ends the turn

An interrupted Claude turn fires NO hook, so the status model ends it from a marker Claude writes into its transcript. This doc covers the measurement, the wiring (`normalizeClaude`, the context tails, both shells' hook-event paths), the `--after` decision, and the residual cases. Main code and tests: `core/claude-turn-interrupt.test.ts`, `createTurnInterruptScanner`, `turnInterruptEvent`, `hook-verified-parity.test.ts`. Fixtures: `shared/agents/__fixtures__/claude/interrupt-capture.json` and `interrupt-queued.json`.

Related: [agent messaging](agent-messaging.md), [context meter](context-meter.md).

## Measurement

Measured on Claude Code **2.1.285**, interactive TUI in a private tmux server, capture hooks via `--settings`, every `NODETERM_*` unset. Esc while it streams, Esc during a foreground tool call, Esc on a permission dialog, and Ctrl+C once mid-stream each fire **nothing**: no `Stop`, no `StopFailure`, no `PostToolUse(Failure)`, and **no `idle_prompt` either**. That notification came 60 s after a NORMAL `Stop` but not in 75 s or 80 s after an interrupt, so the `idle` rescue in `normalizeClaude` does not cover this case.

Before this, a node sat on RUNNING (or NEEDS YOU, for a dismissed permission dialog) until the 20-minute stale sweep: `--after` dependents waited, Eco never saw it idle, the notch and the phone showed it working.

What the interrupt DOES leave is a USER record with content `[{type:'text', text:'[Request interrupted by user]'}]` (`… for tool use]` when a tool call or its dialog was cancelled), whose **`promptId` equals the turn's `UserPromptSubmit` `prompt_id`** in every capture.

## Wiring

- `normalizeClaude` puts `prompt_id` on the `UserPromptSubmit` event as **`turnId`**; the mirror keeps it (`MirrorEntry.turnId`, runtime-only, dropped at a session boundary).
- The claude context tails (local, and the desktop's SSH one) scan COMPLETE lines with ONE stateful scanner per tracked transcript (`createTurnInterruptScanner`): a CLOSED set of the two texts, array content with exactly that one text part, non-sidechain (a typed prompt is a plain string, so typing the words matches nothing).
- **A marker counts for turn P only if P's OPENING prompt record was read BEFORE it** (bounded set of seen prompt ids, 256). The id alone is NOT enough: in real transcripts on the dev host (2.1.209 to 2.1.283), 34 of 114 accepted-shape markers carried the promptId of the prompt written AFTER them. This is "queue a message while Claude works, then Esc": the CLI tags the marker with the QUEUED prompt's id and writes that prompt about 36 ms later, and its `UserPromptSubmit` has already made it the node's current turn, so an id-only match ended the NEW live turn (fixture `interrupt-queued.json`). Measured after the fix: all 26 queued-shape markers rejected, no real interrupt lost. The one interrupt this drops is the one it cannot place; the interrupted turn really ended and the node is already in the next one. The remote tail's historical first read records prompts but never reports.
- **Both shells** check the marker with `turnInterruptEvent` (a mirror PEEK) and push the result through their ONE hook-event path: desktop `emitAgentStatus` (mirror, broadcast, Notch HUD, agent messaging, station notices), Server Edition `emit` (mirror, broadcast, `opts.onEvent`: its delivery queue and `--after` scheduler). Pinned in `hook-verified-parity.test.ts`.
- It ends the turn ONLY when the marker names the node's CURRENT turn (same session, same `turnId`, state working/blocked/waiting). A marker read back from history, one from a finished turn or another session, or one after a restart (no `turnId` then) changes nothing. A prompt event whose `prompt_id` is missing or not a plain token carries `turnId: ''`, which makes the mirror FORGET the previous id.
- The event is an ordinary `done` + `interrupted` (what a `Stop` with `is_interrupt` already produced), UNverified (a transcript read is not a hook POST), so no completion alert, and the question/approval resets apply unchanged.

## `--after` does NOT release on an interrupted turn

Decision (2026-09-30): the person stopped it, usually to redirect it, and the dependent would start on unfinished work (the same reasoning as #521 for an errored turn).

- It is its OWN annotation, `agentStatus.lastTurnInterrupted` (transient; set by an interrupted `done`, cleared by a new turn or a `done` that is not interrupted). It is read by `depSatisfied`, the QUEUED tooltip (`interruptedDeps`), `list` (`LAST TURN INTERRUPTED`; an error outranks it), the canvas's `armedDepSig` (a verdict can clear under a steady `done`, a guessed interrupt then the real Stop, and the launch effect must re-run) and team progress (its own `interrupted` kind, NOT counted as done, so the ring never says "finished" beside a held dependent).
- It is deliberately NOT `lastTurnError`: the TURN FAILED chip, the station-failure notice and issue runs do not treat an interrupt as a failure.
- **The `idle_prompt` rescue does NOT set it** (`recordsTurnInterrupt`): it is flagged `interrupted` only to stay silent, and since `idle_prompt` follows a NORMAL Stop, a rescue means a lost Stop POST on a turn that finished, so its dependents release as before.
- ▶ and `run` still start the dependent.
- The renderer's older keystroke guess (`inferInterruptAfterSettle`, 1.5 s after a lone Esc or Ctrl-C typed into THAT terminal) now records an interrupted `done` too, so a guess cannot release dependents before the marker lands. It stays because it is the only signal for the residual case below.

## Residuals (measured)

- Esc or Ctrl+C BEFORE the first token rewinds the prompt into the input box and writes NO marker (the transcript ends at the prompt record). Only the renderer guess (keystroke in that canvas terminal) sees it; the mirror (notch, phone, Eco's mirror reads, the Server Edition's headless `--after`) keeps `working` until the next hook or the stale sweep.
- Esc "during a subagent": on 2.1.285 the Agent tool launched ASYNC even when asked for a foreground run, so the parent turn had already ended (`Stop`). Esc at the prompt then fires nothing and does NOT stop the child, whose `SubagentStop` and `<task-notification>` arrive as usual. Nothing to fix there; a truly synchronous child being interrupted was not reproducible.

## Surfaces

Server Edition: same core path (its tail and handler); its own headless `--after` still ignores both #521 and this annotation (pre-existing gap). Mobile: gets the `done` through the mirror.

## Device checklist

1. macOS desktop: Esc mid-stream, mid-tool and on a dialog: RUNNING clears within about 1 s, no chime, an armed `--after` dependent stays QUEUED with the interrupted tooltip.
2. SSH node: the same over the remote tail.
3. Server Edition browser tab.
4. A Claude older than 2.1.285: whether the marker text and `promptId` match there is unmeasured (a changed text or a missing `promptId` matches nothing and degrades to the old behaviour).
5. Queue a message while a turn runs, then Esc: the node must STAY working on the queued prompt.
6. The phone's Live Activity ends on the interrupt.
