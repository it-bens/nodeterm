# Quote to chat

"Add to chat" turns a selection in an agent's terminal or ⌘M thread into a markdown quote for that agent. Nothing is ever sent: the person presses Enter. The pure rules live in `renderer/lib/quoteToChat.ts` (`buildQuote`, `planQuote`), the effects in one funnel, `renderer/lib/addQuoteToNode.ts`; the triggers are `nodes/ChatQuoteButton.tsx` and `terminal/useTerminalQuote.ts`. See [chat view](chat-view.md) for the composer and draft the quote lands in.

## One funnel

Every quote into an agent goes through `addQuoteToNode` (its dependencies sit behind `QuoteDeps` / `defaultQuoteDeps`), never through a `sendText` or a draft write of its own. The funnel picks the draft or the pane by what the person can see, never presses Enter, refuses a pane that shows a prompt, a dialog or no CLI (typed text would answer it), and never moves a refused quote into the hidden draft. A new place that quotes into a node must call it.

## The quote

- `stripTerminalControls` runs first and removes escape sequences whole. `sanitizeChatText` alone drops only the ESC byte and would quote a colour change as `[31m`. Then `sanitizeChatText` runs; leading blank lines and trailing whitespace are dropped.
- It is not a full trim: the first line keeps its indentation, so quoted code keeps its relative indent.
- The text is cut at `QUOTE_MAX_CHARS` (4,000 code points). The toast "Quoted the first 4,000 characters" fires only once the quote has landed, because the app has one toast slot and a refusal must not be hidden by it.
- Every line gets a `> ` prefix and the quote ends in one blank line. A selection with nothing left after cleaning adds nothing, silently.

## Which leg

`planQuote` follows what the person is looking at.

- **The node's ⌘M view is open:** the quote is appended to the node's chat draft (`state/chatDraft.ts`, one blank line apart) and the composer is focused with the caret at the end, whatever the agent is doing (a draft types nothing).
- **The terminal is showing:** an unsent paste at the agent's prompt, `pty.sendText(nodeId, quote, { enter: false })` (the path dictation's Insert uses; every backend implements it). The trailing newlines are stripped, because in an app that has not asked for bracketed paste a pasted newline reads as Enter. Residual: such an app also submits at every inner newline. The agent CLIs ask for bracketed paste, and tmux's `paste-buffer -p` frames it. Whether the Windows session host answers `true` for it is an owed device check (see the checklist). Only a literal `true` counts as a landed write; anything else toasts "Couldn't add the quote to the terminal." and returns `nothing`.
- **A refused pane leg is never redirected to the hidden draft:** the person would not know where the quote went.

## What "open" means

`chatOpen` counts a `.term-chat` for the node that is not `[data-chat-readonly]`: a subagent card's thread carries its parent's id, and a closed transcript has no composer.

- While a card modal is open it looks only inside `.kanban-modal`. Otherwise it takes the first match that is not on a board-covered canvas (`coveredByBoard`, `lib/canvasCovered.ts`: the canvas stays mounted under the board, and a focus-mode node above it still counts).
- `focusComposer` uses the same match, so a composer nobody can see never takes the keyboard.
- So a quote from the card modal's terminal goes to the pane even while the canvas node's ⌘M view is open under the board.

## Pane-leg refusals

Each refusal is one toast (`QUOTE_REFUSAL_TEXT`) with nothing typed and the draft untouched. They run in this order:

1. Not an agent node (`not-agent`).
2. The CLI is not in the pane by the store's flags (`agentProcessInPane`: hibernated, paused, dropped, exited): `agent-gone`.
3. `blocked` / `waiting`, or a held question or approval ticket (`held` / `pendingId`): `needs-answer`, because typed text would answer the prompt.
4. **The kernel's say, for every agent:** the same `chatPaneRefusal` call (`lib/chatPaneGate.ts`, `pty.paneOwner` plus the settings' custom agents) that the ⌘M composer and the phone make before they type. The flags are a hint, not proof: Codex and opencode announce no session end, so after a `/quit` the state still reads `done`, and an agent killed without its SessionEnd leaves a shell in the pane under the same flags. A quote at a shell prompt is one reflexive Enter from running each `> …` line as an output redirection that creates or truncates a file. A pane the kernel says the agent left is `agent-gone`; one it could not read is `unverified` ("Could not confirm the agent is running in this terminal.", the composer's wording). Neither is typed.
5. For an agent whose screen is readable (`SCREEN_DIALOG_READABLE`, claude), a `readChatScreen` that the composer's own rule refuses (`claudeScreenBlocksInput`): `dialog`, or `no-prompt` (no input box on screen). The read exists because the agent's own dialogs (folder trust, `/model`) fire no hook, so the state says idle while one owns the keyboard. An unreadable screen is not evidence and types, since the kernel has already placed the agent in the pane.

`working` is allowed (typing into the box while a turn runs is what a person does), and so is a non-empty box (the quote lands at the cursor).

## Triggers

### The thread button

`ChatQuoteButton` offers "Add to chat" for a non-empty selection whose anchor and focus are both inside one `.term-chat__msgs`.

- It is portaled to `<body>` at the selection's last client rect, because a fixed element inside React Flow's scaled viewport would scale with it. Its mousedown is prevented so the selection survives.
- It hides on a selection change, on a scroll of the thread, of a scroller inside it or of an ancestor (not an unrelated scroller: a streaming xterm scrolls all the time), on a wheel, on a resize, on a press anywhere but on the button, and on a focus move to anything outside the thread (the last two are window capture). Those last two exist because the selection can outlive the click that leaves it: xterm's mousedown calls `preventDefault()`, so clicking into another node's terminal keeps the document selection and fires no selection change, and a sloppy-focus dwell moves the keyboard with no click at all.
- It is not offered for a thread `checkVisibility()` says is not rendered, or on a board-covered canvas. A `MutationObserver` on `data-nt-canvas` hides one already shown when a board opens.
- **It owns Escape while it shows and the keyboard is near the thread** (`escapeInScope`: inside its chat panel, on the button, on `<body>`, or on an element holding the thread; a press on the thread's text focuses the nearest focusable ancestor, which on the canvas is React Flow's node wrapper). It listens in the window capture phase and calls `stopImmediatePropagation`: one Escape hides it and goes no further, not to the composer (whose Escape interrupts a turn) and not to the card modal. The card modal also returns early on `chatQuoteButtonShowing()`, because both listen in the window capture phase and registration order decides. With focus anywhere else (another node's terminal) the key belongs to that surface and is left alone.
- `ChatPanel.quoteTargetNodeId`: absent means the panel's own node, `null` means no button (`ClosedTranscriptDialog`).

### The terminal pill

tmux clears its highlight as it copies, so the only moment the text exists in the renderer is the OSC 52 copy (`handleOsc52Write`'s `notifyCopied`, after the write policy allowed it). The other source is the copy-on-select release for a selection xterm owns (Option/Shift drag, a pane with no tmux). Never `onSelectionChange`: search hits and clears fire it too.

- Both feed `useTerminalQuote`, the one piece of glue the canvas node and the card modal's `ModalTerminal` share. `TerminalNode` publishes its sink in a module-level `quoteSubs` map beside `copySubs`, because those handlers outlive a park.
- It offers a `QuotePill` for `QUOTE_OFFER_MS` (6 s; a newer copy replaces it) at the bottom right, never on the agent's bottom-left input line. Unlike the copy receipt it is clickable, and its mousedown is prevented so the terminal keeps the keyboard.
- The offer is gated at the copy on `agentProcessInPane`, enabled only for an agent node (the created agent, or one a hook identified) and never for a hosted read-only seat. It is not gated on `reportsOwnCopy`: claude takes a quote, only its copy receipt stays off.
- `raised` is static per terminal (one notch up wherever the receipt can show), so the pill never drops under the pointer when the 1.6 s receipt (`COPIED_DWELL_MS`) ends.
- The pill does not take Escape (over a terminal, Escape interrupts the turn). Covering the terminal (⌘M face up, the modal's `covered`) withdraws the offer.
- tmux sends OSC 52 to every client of the pane, so a copy in the card modal also offers an invisible pill on the canvas node under the board; it expires.

### A subagent card's thread

A subagent card's thread quotes to its parent (see [subagent visualization](subagent-cards.md#subagent-visualization)). Such a quote is made on another node than its target, and when it lands in the target's pane it says so with the info toast "Added to the agent's terminal." (`addQuoteToNode(…, { crossNode })`, which `ChatPanel` passes for a subagent card's thread or a named other target; a same-node quote gets no toast). The canvas's `nodeterm:toast` listener draws an `info` toast as its fading info strip, beside the `error` banner it always drew.

## Surfaces

- **Desktop and Server Edition:** full (renderer plus the bridged `sendText` / `readChatScreen`).
- **SSH projects:** full (pane writes ride the master).
- **Kanban card modal:** full; its own ⌘M state decides the leg.
- **Relay / team tab:** the ⌘M thread reads nothing there (transcript reads are refused), so there is no thread button. Its OSC 52 copy is refused by `osc52-policy`, so there is no copy-moment pill. An xterm-owned selection still offers one, and that writes to the host's pane through the relay api, the place the person's own keys go, after the host answered `pty.paneOwner` (editor-only, like `sendText`).
- **Mobile:** not available. A follow-up could add a quote action to the phone's chat screen.

## Device checklist

Covered by jsdom only so far:

- (a) Windows session host: `sendText(…, { enter: false })` answers `true` (else every pane quote toasts a failure).
- (b) Claude: a multi-line pane quote lands as one pasted block, nothing is submitted, and the next key continues after it.
- (c) The thread button's place at a canvas zoom other than 100 % and under Liquid Glass (see [appearance](appearance.md#desktop-wallpaper-and-liquid-glass)). A middle-drag pan fires no scroll or wheel event; the press hides the button, so it is not left behind.
- (d) The pill on a Codex node (receipt on, raised) and on a Claude node (receipt off).
- (e) The card modal's terminal pill with the canvas node's ⌘M view open under the board: the quote goes to the pane.
- (f) A relay tab's Option/Shift selection pill writes to the host's pane.

No longer owed: "select in A's thread, click B's pane, Escape reaches B". The press hides the button whatever the selection does, and an Escape with the keyboard outside the thread is never taken. A selection change could still redraw the button while B has the keyboard (Chromium's selection behaviour on that click is unmeasured); it would then not take B's Escape.
