# Terminal emulation

How nodeterm's xterm instances interpret pane output, seed a fresh renderer, send text safely, maintain scrollback and character widths, and adapt IME input. Main code paths: `src/renderer/terminal/terminal-config.ts`, `src/core/tmux-naming.ts`, `src/shared/terminal-width.ts`, `terminal/ime-mode-switch.ts`. tmux session behavior is in [tmux continuity](tmux-continuity.md); terminal teardown is in [terminal lifecycle](terminal-lifecycle.md).

## Ask the emulator before asking tmux

xterm.js is not just a renderer. It parses the pane's output stream, so it **tracks DECSET modes itself** and exposes them as public API (`term.modes`, `@xterm/xterm/typings/xterm.d.ts:1865`): bracketed paste, application-cursor, mouse tracking, origin mode and the rest. We already read one: `term.modes.mouseTrackingMode` decides whether a click means "follow this file link" (`src/renderer/terminal/file-links.ts:341`).

The rule: before asking tmux, ssh or `ps` something about a pane, check whether the emulator already knows it. Facts about *what the app in the pane is doing* (VT modes, alternate screen, cursor shape it asked for) arrive as bytes we already parse. Facts about *the session* (does it exist, foreground process group, which panes are in it) are genuinely tmux's and must be asked. Mixing the two up creates a dependency on a tmux version we do not control. herdr has no version problem here for exactly this reason: it reads `mode_get(MODE_BRACKETED_PASTE)` from its own state machine.

`PtyManager.bracketPasteRequested` is deleted (see its tombstone in `pty-manager.ts`). It asked tmux for `#{bracket_paste_flag}`, a format first shipped in tmux 3.7 (2026-06-26). Ubuntu 24.04 LTS ships 3.4, Ubuntu 22.04 ships 3.2a, Debian 12/13 ship 3.3a/3.5a and Ubuntu 26.04 ships 3.6a. On all of those it expanded to `''`, exactly like a bogus name, so comparison with `'1'` was false for every pane. The bundled tmux did not rescue it: `extraResources` places it under `"mac"` only, and `bundledTmuxPath` is deliberately the last candidate (preferring our binary would pair a new client with the user's older running server, which upstream refuses). On an SSH project it was unfixable from our side entirely: the remote's tmux is whatever the user's server has.

### The emulator is not the answer to the tmux pane's bracketed-paste mode

The `?2004h` a tmux *client* receives is tmux's own paste-through on the outer terminal (`tty_start_tty`, gated on the outer terminfo `BE`/`BD`), not the pane app's request. It arrives about 5 ms after attach and reads true even for a pane running `sleep 30`. It never toggled across pane switches, window switches, reattach or co-attach. A constant is not a signal, so `term.modes.bracketedPasteMode` cannot stand in for the pane's state.

The SESSION HOST is the opposite case (issue #686). On Windows there is no tmux, so nothing sits between the pane's app and the host's headless emulator: a `?2004h` it sees was written by the app itself, exactly the fact `paste-buffer -p` asks tmux for. `HostSession.bracketedPasteRequested()` reads it behind `outputTail`, like `serialize` (xterm applies writes asynchronously, so an early read answers "no" for the turn that just enabled it). `sendKeysWrites` (`session-host/send-keys-delivery.ts`) sanitizes ALWAYS and frames only when the app asked. Separate writes alone can still be read as one burst (#780).

Both native Windows `sendText` and host `sendKeys` execute through `core/settled-text.ts`: capture a baseline, paste without Enter, then poll at 40 ms for at most 15 polls for a changed, stable screen containing the sanitized text. Only then write one Enter, after rechecking liveness, generation and paste mode. Unknown capture, unchanged output or timeout leaves the paste unsubmitted and returns `pasted-not-submitted`, never `true`.

- The discriminant survives IPC/WS and `sendKeysV2`; canvas writes name the partial delivery, trigger runs record a terminal miss (including queue flush), and one-way UI writers raise a visible warning.
- Test success with `=== true`, never truthiness. Do not retry an unconfirmed submit and duplicate the paste.
- Overlapping `sendText` operations on the same pane are refused before input; insert-only, empty Enter and unframed input retain their contracts. A collapsed, hidden or oversize paste may require manual Enter.
- This is an observed-screen heuristic, not an application acknowledgement. Linux fake-PTY tests cover a 150 ms busy reader. Real Windows Codex/Claude, context-link/write, dictation and rename device checks remain required.
- Existing hosts refuse the additive `sendKeysV2` command until they retire. No raw-write fallback or automatic host restart is performed. Legacy clients still use the existing `sendKeys` command.

## Tmux paste delivery

The actual fix predates the probe. From tmux's own man page: *"If `-p` is specified, paste bracket control codes are inserted around the buffer **if the application has requested bracketed paste mode**."* Introduced 2012-03-03 and shipped in tmux 1.7, it is present on every tmux in the field. We do not have to ask whether the app wants framing: we ask tmux to do the framing, and it applies the pane's real state.

Measured on tmux 3.4: framed when the app requested it, unframed when it did not, correct for a non-active pane, and the whole thing in one round trip:

```
tmux load-buffer -b nt - \; if-shell -F -t <target> '#{pane_in_mode}' 'send-keys -t <target> -X
cancel' \; paste-buffer -d -p -r -b nt -t <target> \; send-keys -t <target> Enter
```

`-r` keeps `\n` as `\n` instead of tmux's default `\n` to `\r` rewrite (see `tmux-naming.ts`).

Two measured hazards:

- **Copy mode silently unframes.** With `#{pane_in_mode}` = 1, `paste-buffer -p` delivers unframed (tmux checks the copy-mode screen, not the app), so a user who scrolled the wheel up gets the one-turn-per-line bug. The `if-shell` guard runs `send-keys -X cancel` first, only when the pane is in copy mode, in the same invocation, restoring it.
- **`set-buffer -- "$text"` hits ARG_MAX** around 200 KB. Use `load-buffer -` over stdin. On SSH, pipe into the remote command rather than putting text in argv.

There is no longer a probe or fallback to weigh: `sendText` delivers through `paste-buffer -p` **unconditionally** (the plan builders live in `tmux-naming.ts`). The old two-step path, probe `#{bracket_paste_flag}` then, on a false answer, deliver `line1\nline2\nline3\r`, mangled every multi-line write on a pre-3.7 tmux. It is gone with the probe.

## Seeding a fresh xterm

`attachReplay` / `seedPaint` in `terminal/terminal-config.ts`. A newly mounted xterm is empty. Since tmux paints its own client, there is usually **nothing to seed**.

- **`none`**: the terminal was **parked** (its buffer is still live and correct), or it is a brand-new node with an `initialCommand`. Seeding either would duplicate content.
- **`cold-snapshot`** (`fresh`, reboot/first open): the tmux session is genuinely gone, so replay the persisted `scrollback-store` snapshot with a "session restored" separator.
- **`warm-attach`** (`!fresh`, app restart, tmux still alive): **seed nothing.** tmux is attached to this client: it redraws the visible screen and owns the history under the wheel. A `warm-history` hydration (`transport.captureHistory` to `tmux capture-pane`) was removed because writing into a buffer that tmux then repaints produced black bands and duplicated screens.

The single exception is a **co-attach joiner** (`seedPaint` to `create-screen`). tmux only repaints on SIGWINCH, so a joiner that did not resize never gets a redraw. The screen captured server-side inside `create()` (`PtyCreateResult.screen`) is the only thing that paints it (see [team presence](team-presence.md)).

A co-attach joiner also misses tmux's mouse-tracking modes (`?1000h/?1002h/?1006h`). tmux emits them only at its OWN attach, and neither the `screen` capture (`capture-pane` carries no private modes) nor a SIGWINCH redraw re-sends them, so the joiner's wheel cannot scroll tmux history until a keystroke makes the app re-request mouse. `join()` therefore sets `PtyCreateResult.coAttachMouse` for tmux-backed joins, gated on a REAL tmux client (`tmuxBacked && !sessionHost`, the same local that gates `coAttachAltScreen`/`tmuxClient`; NOT `persistKey`, because a Windows session-host session also carries it and whose joiner then sent wheel/drag to the shell as SGR reports, on BOTH the screen and resize branches). The renderer writes `CO_ATTACH_MOUSE_SEQ` into the fresh xterm in both `ModalTerminal` and `TerminalNode`. tmux is always `mouse on`, so the enable is idempotent. This was the "can't scroll the kanban card-modal terminal until you press a key" bug.

A co-attach joiner ALSO misses tmux's attach-time `\e[?1049h`, and a renderer reload is a joiner too (it re-joins the SAME still-alive tmux client via `join()`, so tmux never re-attaches it). `join()` therefore sets `coAttachAltScreen` for tmux-backed sessions only, gated on `tmuxBacked && !sessionHost`, never plain-shell/session-host, whose normal-buffer scrollback is their only history. The renderer writes `CO_ATTACH_ALT_SCREEN_SEQ` **BEFORE** painting: entering the alternate buffer clears the display, so writing it after would erase the paint. `TerminalNode` skips it once a resync has superseded the seed, since that repaint may already be on screen.

**Known limitation of "never plain-shell":** a REMOTE SSH session on a host WITHOUT tmux (`tmuxOrExplain`'s plain login-shell fallback) is still recorded `tmuxBacked`. Core cannot tell from here that the remote command degraded, so it also gets the alternate-screen switch, `coAttachMouse` and `tmuxClient`'s resync re-apply, hiding that shell's normal-buffer scrollback; detecting the degrade is a follow-up. Without the alternate-screen sequence a renderer reload left every terminal on the normal buffer, piling up to 10k lines of scrollback and forcing a layout per output frame, measured 16.1% versus 7.3% CPU for one terminal at 20 lines/s.

A resync repaint loses the same two modes: `repaintResync`'s `term.reset()` drops ANY terminal, the solo spawn too, back to the normal buffer and clears mouse tracking. Resyncs happen under backpressure, exactly where the alternate screen exists. Every create therefore reports `PtyCreateResult.tmuxClient` (the same `tmuxBacked && !sessionHost` gate, set on spawn AND join), and `repaintResync` writes `CO_ATTACH_ALT_SCREEN_SEQ + CO_ATTACH_MOUSE_SEQ` between the reset and the paint when it is set. Optional on purpose: an older core or relay peer omits it and gets the old behavior, never a guess. The recycle banner ("session restarted by another user") is written AFTER the joiner's seed paint for the same reason: written before the alternate-screen switch, it sat in a buffer nobody could see.

## Scrollback limits

xterm's own scrollback (`xtermScrollback(settings.tmuxScrollback)`, floored at 1000, capped at `XTERM_SCROLLBACK_MAX` = 10000) is kept for sessions tmux does *not* back (a plain shell when tmux is unavailable) and for cold-snapshot replay. It is not what the user scrolls in a tmux session.

An xterm-shaped buffer never takes the raw `tmuxScrollback`. tmux's `history-limit` stores only used cells; an xterm buffer (the renderer, the Windows session host's headless emulator, `NativeWindowsPane`) allocates every line at full width. The Windows session host's headless emulator and `NativeWindowsPane` use the SAME clamp (`src/shared/scrollback.ts`, re-clamped inside the host for older clients), never the raw `tmuxScrollback`: xterm holds every line at full width (cols x 12 B, about 72 MB for 50k x 120 cols per session) where tmux stores only used cells, and nothing above the renderer's cap is ever shown. tmux's `history-limit` keeps the raw value.

## Character widths: one table for every emulator

Every xterm in the stack sizes characters through `activateTerminalWidths` (`@shared/terminal-width`): the renderer's terminals (`renderer/terminal/unicode-width.ts`, which explains the move off Unicode 6) and the Windows session host's headless emulator, which is what a reattaching viewer is repainted from. The renderer's terminals and the session host's emulator must agree with each other, with tmux and with the agent CLIs about cell widths, or a repaint by cursor position leaves stale text behind. Change widths in that one module, never in one emulator, and test them on a real `@xterm/headless` terminal rather than on the provider's return values.

The table is Unicode 11 plus one rule xterm's tables lack: **a one-cell emoji followed by VS16 (U+FE0F) is two cells**, as are keycap sequences (`1️⃣`). The program writing to the pane (`string-width` in every Node agent CLI) and tmux (`variation-selector-always-wide`, on by default) both count those pairs as two cells; xterm counted one, and every repaint by absolute position after such a character landed a column short and left stale text behind. Measured with claude-hud's `⏱️` status line, which read `113` for `11%` and `2h 327` for `2h 32m`.

VS15 is deliberately left alone (`string-width` sizes `⭐︎` by its base, as xterm already does), as are an ASCII base with VS16 alone (`1️` is not an RGI emoji) and the unqualified keycap `1⃣` (one cell, as tmux sizes it; some `string-width` releases say two). Known xterm limitation: in insert mode (IRM) a widened join overwrites the next cell instead of shifting the line. Agreement is the goal, not a particular number.

## IME mode switching

`terminal/ime-mode-switch.ts` adapts xterm 5.5's composition helper after `open()` in both terminal surfaces. Caps Lock must not finalize an active composition: the subsequent native `compositionend` owns that commit. The test executes the dependency's real helper, with an unpatched double-send control. This pins one event ordering, not every native IME. macOS Chinese Caps Lock still needs a device run, including `insertText` and `compositionend` orderings. Revalidate the adapter on xterm upgrades.
