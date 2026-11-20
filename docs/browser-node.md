# Browser node

The `browser` node (`BrowserNode.tsx`) and the `web` node's webview, the rules that keep an Electron `<webview>` guest alive across project switches, browser control geometry for agent-driven pages, and the link guard for rendered Markdown. Main code paths: `renderer/lib/webviewKeepAlive.ts`, `state/webviewKeepAlive.ts`, `main/webview-zoom.ts`, `browser-actions.ts`, `browser-screenshot.ts`, `canvas/VisibleMiniMap.tsx`, `renderer/lib/markdownLinks.ts`. See also [node kinds](node-kinds.md) for the `web` node.

## Browser node

**browser** (`BrowserNode.tsx`), a navigable Chromium browser wrapping the shared
`BrowserSurface` (webview + toolbar); the last top-level URL persists to `data.url`, and the same
surface backs the kanban card modal's browser popup.
**Page zoom is owned by the GUEST boundary, not the canvas DOM** (`@shared/webview-zoom` +
`main/webview-zoom.ts`, 2026-09-20). Measured on Electron 42.10.1 with a physical wheel injection:
Ctrl+wheel over the page arrived in its OOPIF with `ctrl=true`, the host received NO `wheel`
event, and Electron left the factor at 1 until the guest `WebContents`'s `zoom-changed` handler
called `setZoomLevel` (one level produced 1.2). So `.browser-node__view` / the web node body KEEP
`nowheel`, it prevents React Flow from taking a wheel packet over the page, and removing it
cannot make an OOPIF event bubble. Main's one `web-contents-created` listener installs wheel and
Cmd/Ctrl +/-/0 zoom only for `getType() === 'webview'`; the shared `WebviewZoomControls` calls the
same 50%–300% policy for `WebNode` and `BrowserSurface`, which also covers the card modal. The app
does NOT persist zoom in `project.json`: Electron propagates a zoom level by origin, so a per-node
persisted value would make two nodes for one origin fight. Desktop: full; Server Edition:
controls hidden (no Electron guest); Mobile: N/A (no canvas).

## Rules for contributors

**Never unmount, move or re-key a browser/web node's element.** An Electron `<webview>`'s guest
process dies on DOM detach, and a detach includes any `insertBefore`/`appendChild` MOVE of an
attached element, which React performs whenever a kept child's relative order among kept keyed
children changes. That is why webview-hosting nodes render in one stable pool region at the tail
of the `<ReactFlow>` nodes prop (`renderer/lib/webviewKeepAlive.ts`, read its header before
touching the merge, the node array swap in Canvas's load effect, or anything that reorders
nodes), and why a background project's pages stay mounted as hidden ghosts instead of
unmounting. `display:none` is safe (measured: state, scroll and viewport size survive); a reorder
or unmount reloads the user's page and loses their in-page state.

- **A `<webview>` page's wheel never reaches the host DOM.** The guest is an out-of-process frame:
measured on Electron 42, physical Ctrl/Cmd+wheel reached the page and emitted `zoom-changed` on
the guest `WebContents`, while no host `wheel` listener fired. Keep `nowheel` on the webview host
(it protects React Flow routing) and install page zoom in main through `installWebviewZoom`;
removing the class or adding a renderer wheel handler cannot implement guest zoom. The shared
renderer controls call the same `@shared/webview-zoom` policy directly on the attached guest.

## Browser control geometry (#539)

`browser-actions.ts` waits for quiet layout metrics after wheel input and before resolving
click/type targets; the CDP acknowledgement itself is not a rendered frame. Polling has an
initial grace period, a quiet window and a bounded sample count. A continuously moving page
returns a refusal instead of a misleading movement report.

`driveBrowser` permits only one active action per guest session so a click cannot overtake a
scroll's settlement. Revocation during a polling delay aborts without reattaching the debugger.

**`--screenshot --full true` renders the page as ONE viewport** (`browser-screenshot.ts`
`captureFullPage`). `captureBeyondViewport` alone paints off-viewport layers without re-laying the
page out: MEASURED on Electron 42.9.1 against a page scrolled 400px, the sticky header came out at
y=400 and the fixed footer mid-image, the reported "header twice / overlapping" shot. The capture
now sends `Emulation.setDeviceMetricsOverride` (width unchanged, height = content height, capped at
`MAX_VIEWPORT_OVERRIDE` 16384 and said so), settles, captures, clears in `finally`, and puts the
scroll back with one wheel (an override that tall clamps scroll to 0 and clearing does not restore
it, measured); a position it could not restore is in the reply. Rules a refactor must keep:
- **The allowlist admits exactly that shape**, the four keys, bounded integers,
  `deviceScaleFactor: 0`, `mobile: false`, plus a parameterless clear; every other Emulation method
  stays default-denied. No agent value reaches it; a caller-sized viewport (a resize verb, #539's
  feature ask) is a separate decision with its own validator.
- **The override survives `debugger.detach()` and a reload** (measured), so its lifecycle is the
  LEASE's: `BrowserSession.metricsOverridden` goes up when the set is sent and down only on a
  confirmed clear; every detach we make sends the clear first (measured to take effect when sent
  immediately before `detach()`), and after a detach we did not make the FIRST command of the next
  attach is the clear. The screenshot never reattaches a released lease to clear or restore.
- Side effect, stated in the agent text: the page sees one resize, so `100vh` content renders
  taller (one bounded re-measure follows growth).
- Residual, stated: a detach we did not make (not our release/idle/revoke, not page teardown) in
  the sub-second capture window leaves the page tall until the node is next driven. No path we
  know triggers one today (DevTools does not detach the lease, see browser-lease.ts's probe note).
The disposable probe `scripts/probes/browser-539.ts` asserts all of it on a real Electron
(header once at the top, footer once at the bottom, scroll restored, release clears).

`browser` and `open-browser` reject unknown flags through ONE gate, `browserFlagRefusal`
(`core/canvas-control-core.ts`): the Server Edition runs it inside `parseControlRequest`, and desktop
main calls it directly, because that handler never runs the parser (the `--issue` trap, #918's first
version only reached the Server Edition). This does not add resize, console, network or
attribute-reading actions. Server Edition retains its explicit
browser-control refusal; its supported `open-browser` route shares flag validation. Mobile has
no new wire fields or UI; host-side Desktop control inherits the fix.


Browser control must measure settled geometry. A CDP wheel acknowledgement precedes Chromium's scroll frames. Use the bounded settlement helper before reporting movement or resolving pointer targets, and keep actions on one guest exclusive. A CDP viewport override (`--screenshot --full true`) survives debugger detach and a reload: it goes through the lease, which clears it before every detach and on the next attach. Never send emulation around it, and never reattach a released lease to clean up.

## Webview keep-alive across project switches (browser/web nodes)

Issue #301: a project switch used to reload every browser node's page, SPA state, forms, scroll,
websockets gone, because the load effect swaps the whole React Flow node array and an Electron
`<webview>`'s guest process dies on DOM detach (a webview cannot be parked like xterm). The fix
keeps the ELEMENT mounted instead of trying to preserve anything through a remount. The facts it
rests on are measured (Electron 42.x probes + in-app verification, 2026-08-26):

- A guest **survives** sibling insert/remove around its element (React reconciliation of kept,
  order-stable keyed children never touches them), and **survives `display:none`** of itself or an
  ancestor, state intact, viewport size and scroll kept (the guest is NOT resized to 0), repaint
  pixel-identical on reveal, timers running throttled like a background tab.
- A guest **dies** on any DOM *move* (`insertBefore`/`appendChild` of an attached element detaches
  first), taking a full page reload with it. React moves a kept child exactly when its RELATIVE
  ORDER among kept children changes (`lastPlacedIndex`), and React Flow renders nodes in
  prop-array order keyed by id (`adoptUserNodes` rebuilds `nodeLookup` in array order), so the
  merged prop's order discipline IS the feature.

Mechanics (`renderer/lib/webviewKeepAlive.ts` pure + tested, `state/webviewKeepAlive.ts` store,
merged in Canvas exactly like the ephemeral subagent cards, Canvas state, persistence, undo and
the wire never see any of it):

- Every webview-hosting node (`browser`/`web`) renders in ONE stable **pool region** at the tail
  of the `<ReactFlow>` nodes prop, ordered by the pool's entries; entry order never changes while
  an entry lives (append/remove only, `webviewKeepAlive.test.ts` pins the order-stability
  invariant, `webview-keepalive-reconcile.test.tsx` pins the no-detach consequence against real
  React). Visible cost of the hoist: an unselected browser/web node paints above other unselected
  z-0 nodes it overlaps (selection's z 1000 still wins).
- On switch-away the outgoing project's pages become **ghosts**: same node id, `display:none`,
  non-interactive, parked at the origin, `data.ghost` telling the surfaces to route facts at the
  pool (`updateGhostData`) instead of `updateNodeData`. On return the SAME element goes live
  again; `overlayKeepAliveData` folds ghost-time navigations into the loaded nodes inside the one
  `setNodes`, so the `url` prop never moves under the surviving surface (which would navigate it).
- **The merge is keyed on the MOUNTED project (`keepAliveFromRef`), never `activeProjectId`, and a
  mounted entry whose node is missing falls back to its ghost.** Both exist because the pool store
  (zustand/useSyncExternalStore), the ref and `setNodes` do not land in one commit: a switch renders
  interleavings where the id would otherwise drop out of the merged list for one commit, and one
  absent commit is an unmount, i.e. a dead guest ([MEASURED]: the ghost→live direction remounted
  every returning page until the fallback; live→ghost never did). A genuinely deleted node's entry
  is dropped at the deletion funnels (handleNodesChange's `remove`, `deleteNodes`, the peer-mutation
  remove, project deletion/prune), with the next retire as backstop, never by the merge.
- **The minimap must exclude ghosts from BOTH its node list and its bounds lookup** (#850/#786).
  React Flow's MiniMap ignores CSS `display:none`; setting `hidden` on the real node instead
  unmounts the guest. `canvas/VisibleMiniMap.tsx` projects the live flow store into a provider
  scoped to the map, preserving internal absolute positions, measured sizes and the original
  panZoom instance. Only the map's node collections are filtered; persistence and pool lifecycle
  are untouched. The real MiniMap regression tests cover ghost/live transitions, empty bounds,
  removals, grouped geometry and camera interaction. When upgrading React Flow, keep those
  tests: the projection deliberately mirrors the state fields consumed by MiniMap.
  **A pan/zoom frame takes a transform-only fast path**: when `transform` changed and the node
  collections did not, only the camera fields are copied and a full re-projection follows
  `MINIMAP_FULL_SYNC_DEBOUNCE_MS` after the move settles (rebuilding them per frame re-rendered
  every rectangle; measured 402 forced layouts per 20 s pan vs 1 with the map hidden). Identity
  checks alone cannot replace the full path: xyflow's `updateNodeInternals` mutates `nodeLookup`
  IN PLACE and then calls `set({})`, so any update that leaves `transform` alone syncs fully.
- **Memory bounds** (same posture as park/WebGL: a lever must not end live work): a ghost is
  hidden, so the existing Browser Memory Saver discards its guest after `BROWSER_DISCARD_MS`
  unless loading/audible/agent-driven, `onGuestDiscarded` then drops the entry (a husk would hold
  a cap slot). `BACKGROUND_WEBVIEW_MAX` (8) hard-caps live background guests, evicting
  longest-retired first; `activateProject` runs BEFORE `retireProject` on every switch so a
  returning page sheds its background clock before that eviction can pick it.
- E2E-verified under Xvfb (CDP): same webContents across Alpha→Beta→Alpha, typed form text + JS
  state + tick counter continuous, zero reloads; wrapper + webview DOM elements identity-stable in
  both directions. Server Edition: inert (no `<webview>` in a plain browser, ghosts are empty
  husks, nothing to preserve). Mobile: N/A (no canvas).


Keep-alive ghosts must stay mounted, but must not enter minimap geometry. They use `display:none`, not React Flow's `hidden` (which unmounts the guest). Render canvas maps through `VisibleMiniMap`: it filters both drawing and bounds in a minimap-only store while sharing the live camera. A transparent rectangle alone still distorts the map's scale.

## Links in rendered Markdown

**A link in rendered markdown must never navigate the app window.** DOMPurify keeps an anchor's
href as written, and agents write relative links constantly (`[pty-manager.ts](src/core/pty-manager.ts:4100)`).
A click resolved it against the app document: on the desktop another `file://` path, which the old
`will-navigate` guard ALLOWED ("any `file://` is ours"), so the main window navigated to a file that
does not exist and the whole canvas was gone until a reload; in the Server Edition any `<a href>`
click, relative or http(s), navigated the app's own tab away. Two layers now:
- **Renderer, every surface**, ONE delegated, document-level click listener installed at boot
  (`renderer/lib/markdownLinks.ts`, from `boot.tsx`), scoped by `RENDERED_MARKDOWN_CONTAINERS`
  (`.term-md__content`, terminal ⌘M view + editor Preview, `.term-chat__text`, ChatPanel,
  `.sticky-node__md`, sticky notes on canvas AND in the kanban card modal). http/https/mailto →
  `shell.openExternal` (system browser / a new browser tab); `#fragment` and empty href → swallowed;
  anything else → swallowed + an error toast (`nodeterm:toast`, the only kind Canvas renders).
  Local links deliberately do NOT open a file: resolving one needs the owning node's cwd AND its
  filesystem dialect (session source, core platform, SSH/relay, TerminalNode's `pathConvention`),
  which a document-level handler cannot see; a wrong guess opens the wrong file on the wrong machine.
  **A new markdown surface must render inside a listed container**, `markdownLinks.test.ts` fails
  on a component that pipes `renderMarkdown` into `dangerouslySetInnerHTML` outside the list, and on
  a listed class nothing renders any more.
- **Main, desktop backstop**, `decideMainFrameNavigation` (`main/navigation-guard.ts`) allows a
  main-frame navigation ONLY to the entry document itself (scheme + host + decoded pathname; hash
  and query ignored, so reload and dev HMR work); a safe external scheme goes to the OS; everything
  else, any other `file://` path, any other dev-server path, is blocked.

