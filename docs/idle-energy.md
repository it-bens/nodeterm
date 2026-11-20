# Idle energy: an animation is a frame loop, not a decoration

How the renderer keeps an idle or covered window cheap: the animation gates in `src/renderer/styles.css`, `renderer/lib/windowActivity.ts`, `renderer/lib/canvasCovered.ts` and `terminal/board-render-pause.ts`, with the measurements behind them. For the general method of measuring renderer cost see [performance](performance.md).

## The mechanism

A running CSS animation obliges the compositor to produce a frame every vsync (120 a second on a ProMotion display) for as long as it runs, and each frame re-rasters and re-composites the whole window. That cost is paid once for the window, not once per animation, and it does not care whether anybody is looking: an unfocused window that is still visible (a second monitor, half behind an editor) is not `document.hidden` by any definition Chromium uses, so it keeps producing frames at full display rate.

Measured on the Server Edition, 40 terminal nodes on one canvas, headless Chrome, 25 s idle windows, total CPU across every Chrome process (`/proc/<pid>/stat`):

| state | CPU |
|---|---|
| idle, nothing animating | **1.5 %** |
| **one** visible node with the `working` glow | **33 %** |
| five | 66 % |
| twenty | 101 % |
| twenty, but all panned OFF screen | 2.2 % |
| twenty, under an opaque full-screen overlay | 12.8 % |
| twenty, `animation-play-state: paused` | **1.6 %** |
| twenty, `animation: none` + a static opacity | 1.9 % |
| first-run mobile-launch card open (5 animations, no node glows) | 97 % |

Four things to take from that table, because each contradicts a reasonable guess:

- **The step is at the first animation, not the twentieth.** What costs is that frames are produced at all. A gate that covers most of the app's animations buys nothing: one that keeps running holds the frame loop open, while everything the gate does cover still looks correctly frozen. That is why `--nt-anim-state` is applied to every `infinite` animation in `styles.css` and enforced by scan (`styles.animation-gate.test.ts`) rather than applied to the worst few by hand.
- **Offscreen nodes are already free** (2.2 %): Chromium skips raster and compositing for layers fully outside the viewport. A viewport-gated animation would be work with no measurable return. A canvas of 119 nodes costs what its visible animated nodes cost.
- **`paused` is as cheap as `none`** (1.6 % vs 1.9 %), so the gate freezes each animation where it stands instead of snapping it to a resting frame. Nothing moves at the moment focus is lost, and refocusing resumes rather than restarts.
- **The renderer's main thread is idle throughout.** Over the 25 s window with one node pulsing, `Performance.getMetrics` reported 0.15 s of `TaskDuration`, 1 layout and 51 style recalcs, while the renderer process burned 14.8 % and the GPU process 17.9 %. This is compositor and raster work, invisible to every JS-level profiler and to a timer census.

**What the numbers are not.** Headless Chrome here rasters in software (SwiftShader), so the absolute percentages are inflated relative to a real GPU and none is a prediction about macOS. The A/B establishes the mechanism and its direction; the magnitude on a given machine has to be measured there (`powermetrics --samplers gpu_power,tasks`, and Activity Monitor's Energy Impact, which weights GPU use and wakeups heavily).

**Timers are not the problem.** Over the same idle window the renderer fired 56 timer callbacks in 30 s (about 1.9/s: 40 per-node liveness polls at 1/30 Hz, the 2 s terminal-focus mirror, the 30 s host-RAM read) and zero `requestAnimationFrame` callbacks. The default renderer path has no self-perpetuating rAF loop (glyphgrid's parks after 30 idle frames and is opt-in; the dino game's is focus-gated). Before adding a timer gate for energy reasons, measure: at these frequencies a JS wakeup is nothing beside one frame of compositing.

## Rules when you add an animation

An `infinite` animation is a frame loop and runs whether or not anyone is looking. When you add one:

- Give it `animation-play-state: var(--nt-anim-state);` right after the shorthand so it joins the idle-window gate. `src/renderer/styles.animation-gate.test.ts` fails if you forget, because one ungated animation takes the whole win back.
- Prefer a static state to a pulse wherever the pulse is not carrying information the user needs at a glance.
- A dependency's stylesheet counts too: the same test scans `@xyflow/react/dist/style.css`, so an upgrade that adds an infinite animation fails there (see `dashdraw` below).
- Verify the computed `animation-play-state` on a real element, not the presence of the declaration (see Specificity).
- The gate only helps a window nobody is looking at; a FOCUSED window pays for every loop the whole time it runs (issue #1454: four working-subagent edges and a few dots held one at 120 frames/s). So a status indicator pulses a few times and settles (`ease-in-out 4`), and keeps looping only if it must keep pulling the eye (needs-you).
- A mark repeated per row of a list (the sidebar's subagent rows, the board's subagent chip) is static, never a spinner: a fan-out is dozens of rows. `src/renderer/components/SubagentRows.test.tsx` fails on an animation there.
- Every repeating animation must stop under `@media (prefers-reduced-motion: reduce)`. `src/renderer/styles.reduced-motion.test.ts` fails on one that is neither stopped there nor listed in its `EXEMPT` map with a reason (see [Issue #1454](#issue-1454-a-focused-canvas-with-background-subagents)).
- Anything that animates under the kanban board needs the covered override (see [The board gate](#the-board-gate)), and a status animation ON a board card must be bounded to a few cycles, because no gate pauses what is on screen while an agent works for an hour.

## The window gate

`renderer/lib/windowActivity.ts` sets `data-nt-window="idle"` on the document element when the window loses focus or the page hides. `:root[data-nt-window='idle']` flips `--nt-anim-state` to `paused`. The three per-node glows take a static-lit rule instead of the shared pause: pausing freezes a glow wherever its clock stopped, and the glow that says "this agent finished while you were away" must still be on screen when you come back to look for it. `hud.css` is deliberately excluded: the notch HUD's window is never focused, so the shared gate would freeze it permanently rather than while nobody is looking.

## The board gate

The board is the second gate, and it is a separate attribute on purpose. The canvas stays mounted under the kanban overlay (`display: none` would 0x0-resize every terminal into a tmux SIGWINCH), so its glows and pulses keep animating under something nobody can see through (12.8 % in the table above).

- `renderer/lib/canvasCovered.ts` marks `data-nt-canvas="covered"` for as long as a full-page board view is mounted. Mount is the signal: both board views are conditionally rendered, so it cannot drift from the view state the way a recomputed `kanbanOpen` flag can, and it needs nothing from Canvas.
- `:root[data-nt-canvas='covered'] .react-flow` overrides `--nt-anim-state` on that subtree alone. The variable inherits, so that one declaration is the whole gate for every animation reading it.
- **The pinned sessions sidebar gets the same override** (`:root[data-nt-canvas='covered'] .sessions-sidebar`). It sits at z 12, below the board, stays mounted and laid out there, and its working spinners and unread pulses kept the frame loop open under an opaque board (measured, 20 working agents: GPU 4.5 % + renderer 5.7 % to about 0 %). Anything else that animates under the board owes the same line, pinned in `src/renderer/lib/canvasCovered.test.ts`. Anything ABOVE the board (the controls cluster, the pills, a pinned explorer at z 26) is on screen and must not get it.
- It is not a second writer of `data-nt-window` because the two facts are independent (a board on a focused window; an unfocused window with no board), and one attribute with two owners is a race over who clears it.
- The claim is refcounted: React can mount the incoming view before unmounting the outgoing one, and a plain set/clear pair would let that unmount erase the live view's claim.

### The board gate does not stop xterm

CSS reaches the glows, not the terminals. xterm pauses only on its own geometric IntersectionObserver, and a node under the board still intersects. `terminal/board-render-pause.ts` is the pause. Measured before it existed (dev, 12 streaming terminals): board open 36.5 % GPU vs 0.0 % over silent terminals, and `visibility: hidden` on the canvas 35.5 %, no help. The number with the pause is below (a different harness, so not directly comparable).

- Each TerminalNode wraps its RenderService's `_handleIntersectionChange` so xterm is told "not intersecting" while `boardOpen` (Omni included) is up and the node is not the focus-mode node (the focus surface, z 40, sits above the board, z 25). Uncovering replays the real entry through xterm's own resume and then `fullRepaint`. Parsing never stops, so nothing is lost.
- **Pausing the render service is not enough on its own.** Three per-output xterm 5.5 paths run beside it and never ask whether it is paused: `CoreBrowserTerminal._syncTextArea` (every cursor move: a style and layout pass per frame for a hidden textarea), `Viewport.syncScrollArea` (every scroll: `offsetHeight`/`scrollTop` reads around writes, so forced layouts) and `RenderService.handleSelectionChanged` (every scroll, via `SelectionService.refresh`; the DOM renderer answers it with a full `renderRows`, so every terminal without a WebGL context repainted its whole screen behind the board on every line). `board-render-pause.ts` holds all three while covered (an instance property shadows each, and a held call is recorded) and replays each once on uncover: the selection with its latest range (RenderService keeps that range only through this call), then the scroll area as an immediate sync, then the textarea. A covered dispose (a park) replays too, with the scroll-area sync deferred. If you add another per-output xterm call that touches the DOM, hold it there too.
- Measured (Server Edition, headless Chrome, 20 terminals streaming a status line 10 times a second, board open, glass on, CSS animations held): render pause alone GPU 9.3 % + renderer 17.8 %, main thread 14.4 %, about 950 forced layouts per 15 s; with the three paths held GPU 0.4 % + renderer 3.1 %, main thread 1.5 %, zero layouts.
- Only the board holds them: a terminal that is merely off screen on the canvas still runs xterm's stock paths. `board-render-pause.xterm.test.ts` drives a real xterm 5.5, so a rename in an upgrade fails there instead of turning the hold into a silent no-op.
- Never fold the board into `reportVisible` or the WebGL budget's visibility: that verdict drives hibernation and the offscreen teardown, and the board would put the very sessions it shows to sleep.
- Any other "is the board up" predicate must OR in `isGlobalKanbanOpen()` (`boardCoversCanvas` missed the Omni board until the pause landed, the same omission commit 12564a20 fixed for focus restore).

## Specificity is the quiet failure mode

It has already happened twice. The three glows carry their own `animation:` shorthand, which resets `animation-play-state`, so inheriting the variable does nothing for them and every gate has to name them explicitly. A first draft of the covered rule used `.react-flow__node::after`, lost to `.react-flow__node:has(.term-node.working)::after` on specificity, and measured exactly the unpatched number while looking correct in the diff. When you add a gate, verify the computed `animation-play-state` on a real element.

## Bounded status animations

The idle gate only helps an unfocused window, and an agent mid-turn in a focused one kept `nt-working-glow` looping for the whole turn. Measured (production build, M2, focused): one visible working node cost +3 points total CPU and about 25 style recalcs/s for as long as it ran.

- **Working glow**: 4 cycles of 2.6 s (about 10 s), then rests at `opacity: 0.7`, the same static-lit value the idle gate and Reduce Motion already hold it at. The keyframes start and end at 0.7, so the settle is seamless. A new turn re-adds `.working`, which restarts the pulse, and so does anything else that re-applies the animation: a window refocus (the idle gate sets `animation: none`, so lifting it starts the shorthand afresh) and a node remount (a project switch, a park re-adopt) each replay the four pulses. Still bounded every time.
- **Unread glow**: 4 cycles of 2 s, resting lit at `opacity: 0.85`.
- **Minimap beats** (`mm-pulse-soft` / `mm-pulse-unread`): bounded the same way, resting at full stroke.
- **Waiting `--after` rope**: dashed with the hourglass but no longer `animated` (React Flow's `dashdraw`, 0.5 s infinite). An unread node stays unread until someone looks and a wait can last hours, so on a busy canvas those kept the frame loop open indefinitely.

Measured (46-node SSH canvas, 14 unread nodes, 8 waiting ropes, focused window, dev build): idle renderer+GPU about 120 % to about 25 %, and pausing every remaining animation no longer moves it. Pausing any one family alone saved far less (85 to 104 %), which is the first-animation step again.

- **The board card's RUNNING chip** (`.kanban-badge--running`, `nt-pulse 1.1s ease-in-out 4`, ending at the chip's own opacity 1 / scale 1). Looping, one chip held the frame loop open for as long as its agent worked, and the covered gate cannot reach it because the board is what covers the canvas; under Liquid Glass every column's backdrop-filter ran again each frame (measured, 20 RUNNING cards, glass on: GPU 50 % to 17.5 %). The chip mounts when the agent starts working, so each new turn pulses again; `settings.boardRunningPulse` off still holds it still.
- **The subagent and loop card dots, the sessions sidebar's unread check and the working-edge flow** (see [Issue #1454](#issue-1454-a-focused-canvas-with-background-subagents)).

What still loops while focused: the attention glow, its minimap beat and the sidebar's needs-you bell (needs-you is the one state that must keep pulling the eye), the working node header's walking mascot / brand pulse (deliberate character, #1370), the sidebar's working spinner, and progress spinners that exist only while a task the user started runs. The idle and covered gates pause all of them when nobody is looking. `styles.animation-gate.test.ts` pins the bounded shorthands, the resting values and the keyframe endpoints.

## Issue #1454: a focused canvas with background subagents

Parents idle, four async subagents working, two unread sessions: the only running animations were four `dashdraw` edges, four subagent dots and two sidebar checks, and they held a FOCUSED window at about 120 frames/s (reporter: renderer 108 %, GPU 38 %; 64 % / 3 % with them stopped). Reproduced in the Server Edition under headless Chrome (4 subagents, 20 unread rows, focused): GPU 54 to 55 % + renderer 15 to 17 % on `main`, 4 to 5 % + 2 to 3 % with the dots and check bounded and the edge flow bounded to about 4 s (`.react-flow__edge.animated path { animation-iteration-count: 8 }`). The dash stays, so dashed still means working and solid means done; the driven rope keeps its clay colour and weight.

**Reduce Motion covered none of them:** `main` with the OS setting emulated measured 62 % + 22 %. It is now enforced by scan: `styles.reduced-motion.test.ts` requires every REPEATING animation in `styles.css` (infinite, or an iteration count of 2 or more) to be stopped by a `@media (prefers-reduced-motion: reduce)` rule that names it, or to be listed in its `EXEMPT` map with a reason (progress spinners, dead CSS, and four setup-tour scenes that are a stated known gap). It checks xyflow's edge rule separately, because that lives in the vendor sheet.

## React Flow's `dashdraw` is gated in `styles.css`, not by xyflow

Every `animated` edge (working subagent and loop edges, the driven-browser rope) and an animated connection line run xyflow's own `dashdraw 0.5s linear infinite` from a vendor sheet that never reads `--nt-anim-state`, so before the rule it escaped both the idle and the covered gate.

- Measured (dev build, one synthetic edge with React Flow's real classes, idle gate set, 3 alternating 30 s rounds): Liquid Glass 79 % GPU + 36 % renderer and +340 MB GPU footprint; dark 31.5 % + 29 %. With the play-state rule injected over CDP (not the shipped stylesheet) the edge fell to the no-edge baseline of that run, 0.5 % / 3.3 %. The shipped build has not been measured.
- The rule repeats xyflow's selectors verbatim (same specificity, wins on order after the `@import`). `styles.animation-gate.test.ts` scans the vendor sheet for infinite animations, so an xyflow upgrade that adds one fails there.
- A focused window used to pay the full per-edge cost for as long as an edge worked. Since #1454 the flow runs 8 iterations (about 4 s) and then holds the dash, and Reduce Motion stops it outright. The connection line a person is dragging keeps flowing.

## The viewport is never promoted

A `will-change: transform` on `.react-flow__viewport` during pan/zoom was tried (commit 00c9c5fc, measured 41 to 48 % down to 30 to 36 % CPU on 12 WebGL terminals) and removed. The viewport layer spans the whole canvas, and Chromium rasters it at a scale it ratchets up during a zoom and never lowers. Measured on a 46-node SSH canvas (41 terminals, 1470x923 @2x, CDP-driven wheel zoom 0.8 to 0.12 and pans, dev build): with it, 41 to 252 `tile memory limits exceeded, some content may not draw` warnings per gesture round (blank tiles, which users saw as the canvas flickering on zoom) and no CPU gain (about 170 % total during the gesture either way; the scripted gesture itself ran 36 s vs 28 s); without it, 0. The small-canvas gain does not survive a real canvas. `canvas/camera-moving.test.ts` pins the absence.
