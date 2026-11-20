# Window chrome: raising, geometry, tab strip, application menu, theme

The desktop window as an OS object: when it may be raised, how its size and position are remembered, the title bar and Chrome-style tab strip, the application menu, and the base theme tokens. Code is in `src/main/index.ts`, `src/main/window-state.ts`, `src/main/keydown-intercept.ts`, `TabBar.tsx` and `styles.css`. Menu stand-down rules for chords are in [keybindings](keybindings.md).

## Nothing raises the window without a user action

Issue #737, the third in this family (#665 and #702 were canvas control moving the user's view; this one is the OS window activating over another app).

**Cause.** `win.on('ready-to-show', () => win.show())`. `ready-to-show` fires on the first paint of every main-frame navigation, not once per window (measured on Electron 42.9.1: a `webContents.reload()` on a visible window emits it again with `isVisible()` already true), and the crash auto-reload (`render-process-gone` then `webContents.reload()`) is an unattended navigation. So a backgrounded renderer being killed (macOS jetsam on a machine running several agent CLIs at 335 MB to 1.2 GB each) silently raised nodeterm over whatever the user had ⌘Tabbed to. It is `win.once` now.

**The gate must be first paint, not `isVisible()`.** After macOS hide-on-close the window is hidden but alive, and an `isVisible()` gate would `show()` it on a background reload: the same bug inverted.

**Permitted raises are all a click.** They are enumerated with their triggering action in `src/main/window-raise.guard.test.ts`, an allowlist-with-reasons in the shape of `fs-atomic.guard.test.ts`: a notification tap (the one exception the rule names), a Notch HUD row, a Dock activate, a second launch, and a file dropped onto a terminal. Every new raise (`show`/`focus`) needs a user action and an entry in that guard.

- Nothing an agent can do reaches any of them: canvas control, trigger nodes, hook POSTs, relay/pairing/push and browser-drive contain no show/focus/dialog call, and agent confirms are in-renderer `ConfirmDialog`s, never native.
- The drop IPC's `app.focus({steal:true})` is the only renderer-reachable cross-app activation and now carries the same sender guard its two neighbours (`uiShortcutRecording`, `uiTerminalFocus`) already had: a `<webview>` guest is a webContents in this process.
- **Known gap, listed in the guard rather than fixed:** `standing-host.ts`'s `dialog.showErrorBox` for a locked keyring is app-modal, unparented, and raised from the relay reconnect timer. The honest fix routes it to a non-modal in-app surface and owes a macOS check that a sheet on a background window does not activate.

## Background mode and quitting

Closing the window on Windows hides it to the tray (background mode); it does not quit. The relay host, hook server and notifications must outlive the window there as they do on macOS. Any new app-initiated quit (updater-style) must set `quitting` before `app.quit()`, or the hidden window's close handler turns the quit back into a hide.

## Window geometry is remembered

`main/window-state.ts`, `<userData>/window-state.json`: size, position and maximized state, restored at the next launch. Before this the window opened at a hard-coded 1400x900 every time, on every platform, so a user who works maximized re-maximized it on every launch; Electron persists nothing on its own and no platform does it for us.

The module is Electron-free in the `keydown-intercept.ts` shape (pure decisions over plain rectangles, structural window interfaces), so the refusals below can be pressed by a test instead of only by someone with two monitors. `screen.getAllDisplays()` is called at the seam in `index.ts` and its work areas (not full display bounds) are passed in. The refusals are the feature, because each is a way the naive version is worse than the fixed size it replaces:

- **While maximized the size comes from `getNormalBounds()`, never `getBounds()`.** The latter returns the maximized rectangle, so saving it makes the next un-maximize hand back a screen-sized window: state that looks right and behaves wrong, and only for the users who maximize. Un-maximized it is `getBounds()`, the same rectangle wherever both work: Electron documents `getNormalBounds()` as supported only on some Linux desktop environments, and the common case must not depend on the window manager. The maximized case still does and cannot be helped from here, but it matters less, because that record restores by re-maximizing rather than by its size.
- **A position that is no longer reachable is dropped, not clamped.** A laptop undocked from the monitor its window was on would otherwise reopen the app off-screen: running, focusable from the dock, visible nowhere, with no gesture that rescues it. Reachable means a real overlap with some work area (`MIN_VISIBLE_WIDTH` / `MIN_VISIBLE_HEIGHT`), judged against the clamped size; a few pixels on screen is not a title bar anyone can grab. Overlap alone is not enough, because it is symmetric: a monitor mounted above the laptop and then unplugged leaves a record whose bottom edge clips the laptop's work area by enough to clear the height floor while the title bar sits hundreds of px above the screen, and under `titleBarStyle: 'hiddenInset'` the title bar is the whole drag region. So the window's top edge must also land on that work area, within `TOP_OVERHANG_SLACK` (24px, because window managers report decorations inconsistently and a few pixels of overhang must not cost the user their position every launch). The rule is per-display, like the overlap it joins. Dropping keeps the user's size and lets the platform place a window it knows how to place; inventing a corner for it is the guess.
- **No capture while minimized or fullscreen.** `isMaximized()` is false while a macOS window is fullscreen, so capturing there records `maximized: false` and erases exactly the preference this exists to remember. The last non-fullscreen state stands, which also means the app never reopens into fullscreen. That is deliberate: a fullscreen window is usually a temporary mode and is much harder to escape on first launch than a maximized one.
- **Maximize before the first paint**, while the window is still `show: false`; maximizing after `show()` is a visible jump from the restored size on every launch.
- Every field is re-validated as a number on read: the file is hand-editable and its values reach the `BrowserWindow` constructor before there is a window in which to report a failure.
- Saves are debounced (`resize` / `move` fire continuously through a drag) and flushed synchronously on `close`, the only moment guaranteed to see the final state; an awaited write there races the process exit. Published through `renameAtomicSync` with a per-call unique temp, like every other store.
- **`NT_MULTI` is excluded:** a throwaway dev sandbox may share the real app's userData, and it must not move the window of the app being developed.
- Desktop only, and genuinely so: a browser tab's geometry is the browser's, and the mobile companion has no window. Nothing here belongs in `src/core`. **Wayland caveat:** a native-Wayland client cannot set its own position (the compositor owns placement), so x/y is honoured under XWayland and quietly ignored otherwise. Size and maximized restore either way, which is what the drop-don't-clamp rule already degrades to.

## Title bar and drag regions

macOS uses an integrated title bar (`titleBarStyle: 'hiddenInset'`). The tab strip's stationary viewport is the only `no-drag` region for its contents. Explicit regions on scrolled descendants escape the overflow clip in Electron's native hit test and subtract from the wordmark after scrolling (#847). Do not add `app-region: no-drag` to tabs or their descendants: keep tab/button/input descendants at the initial `none`; the viewport already excludes dragging over them.

`scripts/tabbar-drag.test.ts` uses real Electron and native XTest input on a disposable Xvfb display (CDP input bypasses this hit test). It checks 41 tabs at start/partial/middle/end/back, 28/40/64px heights and both padding modes. It does not verify macOS traffic lights, Windows, or movement under a real window manager; those still need device checks.

## Chrome-style tab strip

The bar (`TabBar.tsx`) is the drag region with the `nodeterm` logo plus a Chrome-style tab strip. Inactive tabs are flat and separated by a 1px divider that drops on both sides of a hovered or active tab; hover is an inset pill; the active tab is `--canvas-bg` with rounded top corners and two concave flares (`.tab.active::after`, radial gradients) so it merges into the surface below, which is also the kanban overlay's colour, so it merges under both views. In light the strip steps back to `--surface-deep` (`--tabbar-bg`), because `--panel` and `--canvas-bg` are one value apart there and a canvas-coloured tab would vanish.

**A tab is as wide as its own name and never shrinks** (`max-width: var(--tab-max)` 260px, `flex: 0 0 auto`; the name is `flex: 0 1 auto; min-width: 0` with `text-overflow: ellipsis`). The strip scrolls when the tabs stop fitting, and nothing shortens a name except that cap. This is the deliberate departure from Chrome, which shrinks because it must keep every tab reachable without scrolling: here the sessions sidebar and ⌘1..9 both reach a project without touching the strip, so a readable name is worth more than a visible tab edge.

Two earlier rounds tried to ration the name between tabs, and both spent the one thing the strip is for:

- #789 gave every tab one flex basis. At 8 tabs / 1340px the active name measured 24px (zero characters), because it carried the most furniture and hit its floor first.
- #790 added a 60px floor plus a `tabDensity` level that shed furniture to pay for it. Its middle level hid the SSH chip, and 8 tabs in a 1340px window land on exactly that level, so every remote tab lost its `SSH` label at the width people work at.

`renderer/lib/tabDensity.ts` is deleted: with full names there is no name budget to protect, so a density level buys nothing. Do not reintroduce one without first saying what it is buying. Measured on the third shape (headless Chrome, the real CSS, 86px traffic-light reservation): 8 tabs in 1340px, every name whole, every SSH chip present, the strip 14px past its width; 12 tabs, every name still whole, the strip scrolling 559px; a 60-character project name stops at the 260px cap and is the only thing that ellipsises. The fade mask went with the shrinking: a mask gradient is unconditional and would fade the tail of a name that fits perfectly, while `text-overflow` is self-conditioning.

**The bar's height is one number in two places that cannot read each other:** `--tabbar-h` in `styles.css` (every top-anchored panel, the kanban overlay and the usage popover position against it) and `TABBAR_HEIGHT_PX` in `@shared/window-chrome-metrics`, from which main derives the traffic-light `y` (`trafficLightY`). A literal 15 for a 44px bar was what made shrinking the bar hazardous.

- The default is 40px (36 for one release, which read as cramped once the tabs carried whole names), and the height is a setting: `settings.tabBarHeight` (Settings > Appearance, 28-64). `App.tsx` writes the resolved value to `--tabbar-h` on `<html>`, and main re-centres the macOS traffic lights on the same settings change (`win.setWindowButtonPosition(trafficLightPositionFor(h))`, change-gated), so the two cannot disagree.
- Every reader goes through `resolveTabBarHeight`, which answers the default for a non-number and clamps; the floor is what keeps the 12px lights inside the bar. The stylesheet token keeps the default literal so an un-hydrated renderer draws the default bar, not none.
- `styles.tabbar.test.ts` pins the token to the constant and every dependant to the token.
- The New-project `+` is a sibling of `.tabbar__tabs`, not its last child: inside the scroller it vanished once the strip overflowed (no visible scrollbar to hint it was still there). The wrapping `.tabbar__projects` is `flex: 1` and stays a drag region (not `no-drag`); the pill itself must not be `flex: 1` or it inflates into an empty capsule.

## Application menu

Cmd+M is intercepted in `main/keydown-intercept.ts` (`before-input-event`, installed from `main/index.ts`; otherwise macOS minimizes) and forwarded to the renderer via `app:toggle-markdown`. Cmd+W (`app:close-node`) and Cmd+0 (`app:zoom-actual-size`) are taken back the same way.

The application menu is ours: `buildAppMenu` (`main/index.ts`) calls `Menu.setApplicationMenu` and re-runs on every settings change. Check the template, not Electron's defaults. Which accelerators are handled above the page, how the menu stands down for the terminal-first policy and an armed shortcut recorder, and why Reload is the named exception are all in [keybindings](keybindings.md).

## Theme

macOS dark palette as CSS tokens in `styles.css` `:root` (`--accent` is systemBlue, label/separator opacities, SF font stack). The canvas background is black with a dot grid.
