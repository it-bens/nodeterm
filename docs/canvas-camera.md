# Canvas camera: selection, zoom, go-to-node, breadcrumbs, directional focus

How the canvas moves the viewport and who may move it: undo and selection basics, zoom chords, per-terminal font zoom, the one camera-travel path (`goToNode` / `frameNode`), the breadcrumb trail, directional focus and the command palette. Code is in `Canvas.tsx`, `renderer/lib/nodeFocus.ts`, `renderer/lib/zoomShortcut.ts`, `renderer/lib/breadcrumbs.ts` and `lib/directionalFocus.ts`. Layout transforms that call `setViewport` after a rearrangement are in [canvas layout actions](canvas-layout-actions.md).

## Undo, selection, delete

- **Undo/redo**: debounced snapshot of the nodes array on settle (drag/edit), `pastRef` / `futureRef` stacks, ⌘Z / ⌘⇧Z and dock buttons. History resets per project load and is skipped while typing in inputs or terminals.
- **Selection/pan**: box-select on left-drag (`SelectionMode.Partial`, touch to select); pan is middle-drag or trackpad two-finger (`panOnScroll`, `zoomOnScroll:false`); pinch zoom. Right mouse is free for the context menu.
- **Delete** (Delete/Backspace) opens `ConfirmDialog` before removing selected nodes.

## Never frame automatically with `fitView`

React Flow's `fitView` is queued, not immediate. Calling it sets `fitViewQueued` and the fit runs from a later `setNodes` (only once every node is measured, `nodesInitialized`) or the next `updateNodeInternals`, against whatever `nodeLookup` holds by then. Its fit set is also filtered by `measured` (no `width`/`height` fallback in `getFitViewNodes`), so a set that comes out empty collapses the bounds to `{0,0,0,0}` and `getViewportForBounds` parks the canvas origin in the middle of the screen at max zoom. Compute the viewport yourself and apply it with `setViewport` (`renderer/lib/nodeFocus.ts`, `canvas/fit-view.ts`), which lands now and against the canvas you meant.

`fitAll` is the one deliberate exception: an explicit user gesture on a settled canvas whose fit set is every node. It carries the same deferral, so do not reach for it from anything automatic.

## Zoom chords

`renderer/lib/zoomShortcut.ts`: ⌘/Ctrl+0 runs `zoomTo100` (actual size, what the browser and Electron's default View menu already mean by that key) and Shift+1 runs `fitAll` (the Figma/tldraw/Excalidraw "zoom to fit"). Matched on `e.code`, like the project-jump chord, which excludes `Digit0` so the two can never collide.

- The module is a pure decision because both chords move the camera, and a camera move here is not read-only: `onMove` calls `markDirty`, which persists the viewport and casts it to the team session. So it refuses while the kanban board is up, while focus is in a text surface (input/textarea/contenteditable/Monaco/xterm, where Shift+1 is just the `!` key), and on auto-repeat (both actions animate; a held chord would restart the tween).
- Desktop ⌘0 does not arrive as a keydown: the default menu's `resetZoom` accelerator wins, so `main/index.ts` intercepts it in `before-input-event` and forwards `app:zoom-actual-size`, which re-asks the same refusals. The Server Edition needs no intercept (no menu; Chrome and Firefox hand ⌘0 to the page) and stubs the subscription. See [keybindings](keybindings.md) for how the intercept layer works.

## Per-terminal font size

`renderer/terminal/terminal-font-zoom.ts` (issue #915). Opt-in via `settings.terminalFontZoomKeys` (Settings > Terminal, default off). On, with a terminal focused: ⌘+ / ⌘- (Ctrl off-mac; exactly one primary, since Ctrl+- on mac is readline undo) step that node's `data.terminalFontSize` by 1 within the global field's 8-28 range, and ⌘0 clears it back to the global `fontSize`. `+` and `-` match on `e.key` (German `+` key), `0` on `e.code` like the canvas chord. Before #915, ⌘+ / ⌘- did nothing (no menu zoom roles, React Flow's key zoom is off; only webview guests zoom on them) and ⌘0 in a terminal was claimed by main and then refused. Off, or with no terminal focused, every key behaves as before.

- The override is persisted per node (`normalizeTerminalFontSize` on both sides of `nodeStatesToFlow` / `flowToNodeStates`) and layered by `useXtermVisualSettings(projectId, fontSizeOverride)`, the same path for the canvas node and the card modal (`ModalSpawn.terminalFontSize`); the settings preview passes none.
- Font size is cell geometry, so `applyLiveOptions` reports `metricsChanged` and both surfaces re-fit and report the new grid exactly as for a global font change.
- **One writer:** both xterm key handlers and the forwarded desktop ⌘0 (resolved from focus via `data-font-zoom-node` on the xterm host) dispatch `nodeterm:terminal-font-zoom`; Canvas applies it (`nextTerminalFontSizeOverride`, `markDirty`). The toolbar and dock +/- buttons stay canvas zoom (they have no focused terminal to act on).
- Main forwards ⌘0 with `{meta, control}` and the renderer resets only on the platform chord (`forwardedResetMatches`), so a mac Ctrl+0 keeps its old meaning.
- Under `terminalGpuRendering: 'shared'`, a node whose size differs from the global one is held off the shared glyph canvas (`leavesSharedGlyphAtlas` to `glyphOff`) and paints its own pixels, because the shared atlas is rasterized for the global font and a grid's cell is fixed at `register`.
- Every xterm is built from the effective visual (`visualRef`, both surfaces): a refresh or offscreen revive recreates it in the same mount, where the live-options effect does not re-run.
- Keypad 0 resets only when it types `0` (with Num Lock off it is Insert, and Ctrl+Insert copies).
- Canvas also mirrors the step into the projects store for the active project (`patchStoredFontSize`, same epoch guard as `commitActiveToStore`), because the Omni board builds its card modal's spawn from the store and would otherwise lag until the next autosave commit.

## "Go to node" (`goToNode` to `frameNode`)

The one camera-travel path: notification click, sessions sidebar, ⌘K jump, presence travel, minimap double-click, double-click focus. It computes the viewport itself and applies it with `setViewport`. It must never go through `fitView`.

**The failure it prevents.** On a canvas whose nodes sit thousands of px out, a `fitView` landing in the unmeasured window gave the field report verbatim: right project, empty stretch of canvas, sometimes. Cross-project focus lands in that window every time (switch project, load its nodes, frame the target, all before the mount-time measuring has settled), and the second click always works, which is what makes it read as intermittent.

**Geometry** is `renderer/lib/nodeFocus.ts`: the node's rect from React Flow's own measurement when it has one (`getInternalNode`; `measured` reaches our node objects one render later via `onNodesChange`, and `internals.positionAbsolute` also accounts for `extent:'parent'` clamping), else `nodeFitRect` from the persisted size, resolving the group-parent chain. Then `viewportForRect`: centred in the pane, and nothing else.

- Framing a single focused node against the chrome-free rectangle was tried twice and is wrong both ways: centred in that rect ("too far right on an ultrawide, half off-screen on a laptop"), and centred in the pane then nudged clear of it ("still not in the middle"). `.sessions-sidebar` is a 300px absolute overlay and it is open exactly when this feature is used, so either rule pushes the node right by most of its width. The couple of dozen pixels that end up behind the sidebar cost far less than losing the centre.
- The free-rect solve (`solveFitFrame` / `solveFitPadding`) stays where it earns its keep: `fitAll`, which fits every node and would otherwise tuck them under the dock.
- Unknowable size or no pane: the camera stands still.

**Exception 1: a maximized node (issue #743).** A node with `data.premaxRect` (`isMaximized`; the flag `maximizeNodeToRect` writes and `restoreMaximizedNode` clears) is framed against the same rectangle `maximizeTargetRect` placed it in, through `viewportForNodeFocus`, which passes `measureMaximizeInsets(box)` to `viewportForRect` only for maximized nodes. `nodeFocus.policy.test.ts` exercises this shared Canvas decision for ordinary, maximized and restored nodes in both zoom modes. Test the maximized-only focus decision through `viewportForNodeFocus`, the helper Canvas calls, rather than passing preselected insets straight to the geometry function.

- Maximize also measures `.controls-cluster` and `.dock` (#711): their screen rectangles reserve top/bottom space with an 8px gap, consuming the existing 24px margin first. Chrome outside the horizontally usable area contributes nothing. Menus and hover peeks never reserve a band. Both focus zoom branches use these same vertical insets; refitting observes the persistent chrome as well as pinned panels and compares all four insets. Zone snap retains its side-only policy. Ordinary focus and zone snap keep their own policies.
- Maximize placement and refocusing must use the same measured usable rectangle (`measureMaximizeInsets`): pinned side panels plus persistent top controls and bottom dock. Do not hardcode chrome heights or add the outer margin twice; transient menus must not resize terminals.
- The trade-off above rests on one number, how much of the node ends up behind the panel. For a maximized node that number is set by the panel rather than the node, by construction: maximize sized it to be exactly the free area, so centring it in the wider pane buries half the inset less the margin. Measured by the reporter on a signed v0.3.5 build with the sidebar pinned: an ordinary node lost 33px, the maximized one 137px, and the camera drifted by `322 / 2 = 161` px on every "go to another node and back".
- Because the node is that rectangle minus two margins, centring it in the free area reproduces maximize's own origin (`marginPx + insets.left`) exactly, which is what makes this a fix rather than a second opinion about placement. It applies to both zoom branches (the rectangle question is the same one; splitting it would be two rectangles again, which is the bug). With no pinned panels or overlapping persistent controls, `insets` is zero. A pane narrower than the panels over it falls back to the whole pane rather than solving against a negative width. `measureMaximizeInsets` reads the DOM only when framing a maximized node.

**Exception 2: a pinned side panel (issue #854).** An ordinary node is still centred in the pane, but if that lands part of it under a pinned sessions sidebar or explorer (`measurePinnedInsets`: pinned only, never the hover overlay), `clearOfPinnedPanels` moves it the least distance that uncovers it (plus a 12px gap when there is room). A node too wide for the free area stays centred at a caller's zoom (a shift only swaps which edge is covered) and is fitted into the free area when the zoom is ours. This is not the rejected nudge of commit 5e8abfe7: that cleared the sidebar as an overlay, which is open whenever the click comes from it, so every jump moved. Pinning is the user's statement that the panel stays. The reporter measured 36-47px of a large node under a pinned 321px sidebar on every jump.

**Zoom setting.** `settings.focusZoomToNode` (Behavior, default on) is the escape hatch for the rescale: off, the camera keeps the zoom `getZoom()` reports and only pans, and that zoom is passed through unclamped. It is one the canvas is already displaying, and re-clamping it to the framing range would rescale the view the option exists to leave alone.

**Two rules a refactor must not undo.** Framing goes through `frameNode` and nothing else (`canvas-wiring.test.tsx` pins that it contains no `fitView(`). And a "stands still" branch must never be "helpfully" replaced by a bare `fitView`: that is the origin jump.

## Breadcrumb trail

`renderer/lib/breadcrumbs.ts` holds all the pure logic. Every deliberate `goToNode` landing records a `NavStop` (`{nodeId, at, note}`) for the active project. ⌘[ / ⌘] (`canvas.goBack` / `canvas.goForward`, bound in `shared/keybindings.ts`) and the two Dock buttons walk that trail. On a project activation a once-per-app-run `ResumeCard` offers the last few distinct stops ("resume where you left off"). The card is opt-in via `settings.showResumeCard` (Settings > Appearance, default off); while disabled the once-per-app-run slot is not spent, so enabling it later still shows the card on the next activation. The chords and Dock buttons work regardless.

- **The trail is machine-local and rides `IndexEntryV3.breadcrumbs`, never `.nodeterm/project.json`**, the same tier as `viewport` / `defaultAccountId` / `capabilityAck`, for the same reason: a repo must not carry one person's camera history to everyone who clones it. `fileToProject` therefore ignores a `breadcrumbs` field found in the shared file (a forgery), and `projectToFile` never writes one.
- **The cursor is not persisted either.** Only `list` rides the entry; `BreadcrumbState.index` is renderer-only and resets to the tip on activation. A step records no breadcrumb and rewrites no `project.json`; the only persistence it triggers is the ordinary `onMove` viewport persist (machine-local, same as any camera move).
- **Cap 20** (`BREADCRUMB_CAP`, oldest dropped) and a 3 s dedupe (`BREADCRUMB_DEDUPE_MS`): a re-triggered focus on the already-current node is a no-op, and `recordBreadcrumb` returns the same object, which is the caller's skip test. Recording past a back-step drops the forward tail, exactly like a browser tab.
- **`stepBreadcrumb` skips stops whose node is gone** (never lands on a dead entry; no reachable stop gives `null`, and the camera stands still), and `goToNode` refuses to record ephemeral `subagent` / `loop` nodes. They are merged into the `<ReactFlow nodes>` prop but never persisted (cleared on the next turn), so a breadcrumb for one is an id nothing can ever resolve, burning a slot forever.
- The `note` is a snapshot taken at record time (agent nodes reuse the sessions sidebar's own `sessionStatusKind` + `STATE_LABEL` phrasing, preferring session name, then node title, then agent label), so a later state change never retroactively rewrites history.
- **Surfaces:** the Server Edition works as-is (shared renderer code plus `WorkspaceStore`, which both shells boot; no new bridge member). Mobile N/A (no canvas, no camera). The kanban board is likewise N/A, and a project that activates on the board neither shows nor spends its once-per-run resume card (it would sit invisible under the opaque overlay).

## Directional focus

⌘←/→/↑/↓ (`node.focus*`) is the multiplexer gesture (Ghostty `goto_split`, tmux `select-pane -L`) on a canvas with no split tree. `lib/directionalFocus.ts` ranks candidates by geometry (same-row/column overlap first, then diagonal with a sideways penalty; no wrap). The one decision, where to land and how, is the pure `planDirectionalMove`, and the "how" has two answers that must not be merged:

- **On the canvas the move travels** (`focusNodeById`: frame, keyboard, read-ack). The origin is the node the keyboard is in, else the selection, else the first press adopts the node nearest the view centre.
- **In focus mode (⌘⇧F, issue #78) the move swaps the focused node** (`useFocusNode.focus(next)`). The flow pane is `display:none` there, so a travel would frame a node on a 0-sized pane (the origin-jump bug) while the user kept staring at the old terminal. Before this, moving between two focused terminals was ⌘⇧F, ⌘→, ⌘⇧F.

Three details carry the focus-mode leg: the origin is the focused id (the reparent detaches the node root from its `.react-flow__node` wrapper, so the keyboard origin reads null there); candidates are terminals only (the one kind focus mode can show, `focusTargetId`); and the keyboard is handed over explicitly (`useTerminalFocus.request`), because the reparent gives focus back to whichever xterm had it, i.e. the old node. Selection follows the swap so leaving focus mode lands on the node the user was in. An edge with nowhere to go is a claimed no-op in both modes: declining would hand ⌘→ to the terminal, which is not "nothing".

## Command palette

`CommandPalette.tsx`, opened with ⌘/Ctrl+K. `Canvas.buildCommands` builds the commands (create, switch project, jump to node by title/tag, open file, and so on).
