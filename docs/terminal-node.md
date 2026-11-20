# Terminal node

The `terminal` node kind (`TerminalNode.tsx`) and the agent preset built on it: header and hover guard, the markdown view, selection, copy and paste behaviour, link handling, and the overlay rules that keep terminal wheel and pointer routing intact. For the tmux session model underneath see [tmux continuity](tmux-continuity.md) and [terminal lifecycle](terminal-lifecycle.md); for xterm emulation details see [terminal emulation](terminal-emulation.md).

## Header, hover guard and the markdown view

**terminal** (`TerminalNode.tsx`), xterm + tmux (see [tmux continuity](tmux-continuity.md)). Header: collapse, color,
click-to-rename title, ✦ AI-name, ×. Body has a **hover guard** overlay: dwell
`settings.panHoverDelay` (default 600 ms) before the terminal takes focus, before that,
drag = move node, scroll = pan canvas. **Cmd/Ctrl+M** (while hovered) toggles a markdown
render of the captured output, `nodes/TerminalMarkdownView.tsx`, which takes a node id + a
`capture` function (no React Flow context, so the kanban card modal can reuse it) and owns the
lifecycle: capture on mount + a ↻ refresh, a request token that drops stale/late answers, an
explicit empty state ('' is an answer; a rejected capture is a failure, not empty), only the
LAST `MD_OUTPUT_MAX_LINES` (5000) lines rendered and announced when cut, scrolled to the latest
output. Entering the view (output or ChatPanel) blurs the xterm and, for the ChatPanel, puts the
keyboard in its composer (only when focus was nowhere, in this node or in a terminal, never out
of a text field elsewhere; a lazy-loading or disabled composer takes it when it mounts/enables,
and an exit cancels the request); leaving it restores focus
only if the terminal had it on entry AND focus is now nowhere or still inside this node
(`terminal/useMdModeFocus.ts`). The open view is PERSISTED (`CanvasNodeState.mdMode`, written only
when open), so a project switch or a reload brings the node back on the view it was left on.
While the view is open, every "take the keyboard" path (hover
dwell, click, sidebar/notification jump) goes through `focusXtermUnlessCovered` and leaves the
hidden xterm unfocused, the overlay sits inside the node body, so a dwell over it used to route
keystrokes into a pane nobody could see. The body's file DROP / file PASTE handlers (which focus
the xterm and paste paths into it) are the other way in, and they stand aside while covered
(`terminalOwnsFileInput`): a screenshot pasted into the ChatPanel composer used to be caught in
the capture phase and typed as a path into the hidden pane. Both are source-pinned in
`useMdModeFocus.test.tsx`. Tag chips via `NodeTags`.

## Selection and copy

**Selection + copy is tmux's** (its mouse is on, see [tmux continuity](tmux-continuity.md)): drag to select, wheel to
scroll tmux's history. A drag copies via copy-mode, and tmux emits **OSC 52** to the client, whose
handler writes the **system clipboard**, the one copy path on every platform *and* over SSH (no
`pbcopy`). OSC 52 writes an app emits itself (vim `"+y`, gh, yazi) reach the clipboard through the
same handler (write-only, a read query is refused). The emulator's own copy chords stay for a
selection xterm *does* own (`copyKeyAction`/`isCopyShortcut`): **Cmd+C** (mac), **Ctrl+Shift+C**
and **Ctrl+Insert** (Linux/Windows), matched on `e.key` *or* the physical `KeyC`, so non-Latin
layouts still copy. A copy chord is **always swallowed**, selection or not: letting Ctrl+Shift+C
fall through would reach the pty as `\x03` (SIGINT). Ctrl+Insert exists because Chromium reserves
Ctrl+Shift+C for the inspector and a page cannot `preventDefault()` it, which is where Server
Edition users land. Plain **Ctrl+C** is never intercepted.

## Text paste

**Text paste uses the platform event** (`isPasteShortcut` → the `'native'` action): ⌘V on
mac reaches the Edit menu's `{role:'paste'}`, whose `paste` event xterm frames
as a bracketed paste. All the terminal does is stop CANCELLING the chord, and that is a
**Windows-only** claim: xterm's keymap turns Ctrl+V into `\x16` with `cancel`, which suppressed
Chromium's paste command *and* the Ctrl+V accelerator behind it, so Ctrl+V pasted nothing at all
there (issue #562). Off Windows the chord stays `\x16` on purpose, mac pastes with ⌘V, Linux
with Ctrl+Shift+V, and Ctrl+V is a key vim/readline users really send. Ctrl+Shift+V and
Shift+Insert need no branch: measured against `evaluateKeyboardEvent`, xterm produces neither a
key nor a cancel for them, so the platform already pastes. To select in **xterm** instead of tmux
(or inside an app that grabs the mouse, like vim/htop), hold **Option** (mac,
xterm's `macOptionClickForcesSelection`) or **Shift** (Linux/Windows) while dragging.

## Screenshot paste

**Screenshot paste (#712):** the capture handler in both TerminalNode and ModalTerminal
owns files/images: save/upload, then paste the path, suppressing accompanying text. A
macOS Ctrl+V may instead let a local foreground agent read its own system clipboard.
Configured agent identity proves neither foreground state nor clipboard support, and a
PTY write has no image receipt. Never synthesize that key or fall back between routes.
The macOS shortcuts reference explains both keys; its Server Edition copy explicitly
says Ctrl+V cannot transfer the viewer's clipboard to the host. SSH keeps remote uploads.

Keep shell, SSH and Server Edition file routing and the suppression of accompanying text. Do not synthesize Ctrl+V or try both routes without a capability and receipt protocol. The shortcuts panel documents the Cmd+V / Ctrl+V distinction.

**The path paste has a receipt for claude, and only for claude** (`terminal/image-paste-confirm.ts`,
both surfaces through `pasteWithImageReceipt`). MEASURED on Claude Code 2.1.285 (bracketed paste,
captures in `terminal/__fixtures__/claude-image-paste.json`): a path to an existing
png/jpg/jpeg/gif/webp (any case) becomes `[Image #N]` in the composer within ~60 ms; bmp, svg,
heic, tiff and a missing file stay text; `N` keeps counting for the session and does NOT reset
when the composer is cleared. So the receipt reads OUR xterm buffer (the emulator, not tmux) for
placeholder numbers ABOVE the highest one on screen before the paste (the counter only rises,
so an older placeholder scrolling into view cannot confirm it; with none on screen, two pastes
inside ~60 ms can still confirm each other), up to 3 s: "Image attached", else
"Pasted the path, not confirmed as an image" (`.term-paste-pill`, top-right so it never covers
the copy pill or the agent's bottom-left input line). Only when a claude CLI is in the pane
(`agentProcessInPane`); every other agent was not measured and gets the paste with no receipt
either way, nothing is claimed on its behalf. A terminal disposed mid-wait reports nothing.

## Copy feedback

**Copying now says so**: the OSC 52 handler floats a transient `Copied N lines` pill over the
terminal's BOTTOM-RIGHT corner (`.term-copy-pill`, the same class on the canvas node and the
kanban card modal, one session seen twice must not speak in two voices; bottom-right because
every agent CLI writes its input line bottom-left, and `pointer-events: none` because it sits on
the terminal and fires on every copy), because tmux's `copy-pipe-and-cancel`
clears the highlight at the exact instant it copies, which read as "the copy failed" to a user
whose other pane ran claude. And a drag that produced NEITHER an OSC 52 nor an xterm selection
means the pane's app captured the mouse (claude does, codex does not), so a one-time
`Hold ⌥ to select text` hint fires instead (`nodeterm.seenSelectHint`). **The whole layer is
OFF for an agent in `SELF_REPORTS_COPY` (`reportsOwnCopy`, claude, which prints its own
"copied N chars to tmux buffer" line): a second message for one gesture is noise, and a claude
terminal is byte-identical to before the feature. **One owner per pill:**
the `copied` receipt is raised ONLY by the OSC 52 path, the hint ONLY by the drag path, the two
never race for the same slot. The emulator's own copy **chord** (Cmd+C / Ctrl+Shift+C) deliberately
raises nothing: Claude Code prints its own copy line ("copied N chars to tmux buffer"), and a
second message for one gesture is noise. Decision logic is the pure `terminal/copy-feedback.ts`;
`useCopyFeedback` is the glue (it also yields to a clipboard-failure `nodeterm:toast`, so the
Server Edition never shows a green receipt beside a red banner), and the node publishes its sink
through the module-level `copySubs` map because the OSC handler survives a park.

## Shift+Enter and links in the output

**Shift+Enter** is remapped to `\x1b\r` (ESC+CR / M-Enter) so agent CLIs insert a newline
instead of submitting (`terminalKeyAction` / `SHIFT_ENTER_SEQ` in `terminal-config.ts`; sent in
all terminals, harmless in a plain shell). **Cmd (mac) / Ctrl+click** opens links in the
output: URLs → default browser (`@xterm/addon-web-links`), file paths → editor node and
directories → Explorer reveal (`terminal/file-links.ts`, existence-verified against the project
fs via cached parent-dir listings, with `path:line[:col]` compiler-output suffixes). **What counts
as a path is `terminal/file-link-tokens.ts`**, and it is generous on purpose because existence is
the arbiter: segments take any Unicode letter/number/mark (`var/otta-aktarım/çıktı.sql`) and
route-folder brackets (`app/(shop)/[id]/page.tsx`; prose parentheses are dropped only when
unbalanced); a separator path with SPACES is offered whole, ending at a word with a separator, a
word completing `name.ext`, or the line end, AND as its space-free pieces, which is what keeps
`/usr/bin/python failed to start app.py` from costing the `/usr/bin/python` link (tokens may
overlap; the provider keeps the first that exists in start/longest order, `linkAtCell` carries the
rest as `alternatives` for the Cmd+click fallback and the link menu); a BARE filename (`README`,
`foo.ts`) only when `looksLikeBareFilename` says so, versions (`v1.2`), abbreviations (`e.g.`)
and plain words are never looked up, and a right-click does not claim a bare word; a `file://`
URI is percent-decoded to its absolute path (local host only; Windows needs a drive). **The scan
must stay linear**, it runs per hovered row on padded TUI rows; no backtracking regex, and the
ReDoS guard in `file-link-tokens.test.ts` pins it (the previous regex took 2.2 s on a 30k-char
word). Hit-tests and underlines go through `Paragraph.cellStart/cellEnd` (xterm cells, not string
indices), so wide CJK glyphs before or inside a path no longer shift its range. A relative
path is anchored on the node's LAUNCH cwd first, then on the pane's LIVE cwd (`pty:pane-cwd`,
tmux `#{pane_current_path}`, local or over the ControlMaster; `findExistingPath`), because an
agent prints paths relative to where IT runs; the launch cwd wins a tie so a link never changes
meaning when the pane moves. A Cmd/Ctrl+click on a path that exists under neither raises a
`File not found: …` toast naming where it looked, the click is swallowed before the async
lookup, so without it the gesture silently did nothing. **"Could not check" is never "not
found"**: a lookup has three outcomes (`PathLookup.unverified`, `PathResolution.unverified`), and
`makeDirListingLookup` treats a REJECTED listing and an EMPTY one as unverified, `FsApi` is
fail-open (`listDir` ends `catch { return [] }`, and a dead ControlMaster lists `[]`), so only a
listing WITH entries can prove absence (same rule as `classifyEmptyListing`); `.git` is unverified
too (both listing legs strip it). A failure is cached as a failure for ~1 s, never as an empty
directory for the 3 s TTL. An unchecked launch-cwd candidate still lets the live cwd be tried (a
hit there is proof). The toast then reads `Couldn't check <path>: <reason>` (`fileMissMessage`),
and the link menu shows `Couldn't check: <reason>` instead of `Not found`. The path
dialect follows the FILESYSTEM-OWNING CORE, not the viewer: desktop-local may use its own
platform, Server Edition and relay tabs use the core's reported `process.platform`, and SSH
projects are POSIX. A failed host-platform read disables file links for that connection, it
never guesses from the browser. Standalone `ssh` terminal nodes remain URL-only because they
have no remote fs API with which to verify a token; relay tabs do have a core-bound, jailed fs
API and therefore support file links. Windows existence matching is case-insensitive and accepts
both separators; UNC tokens are refused whole before they can be reinterpreted as cwd-relative.

## Dropped and pasted file paths

**Dropped / pasted file paths** follow the same core-platform rule (`dropDialect` /
`quoteDroppedPath`, `terminal/file-drop.ts`): POSIX backslash-escaping everywhere, every SSH
pane and Git Bash included, byte-identical to before, except a Windows core, where `\` is the
separator and is never doubled: bare when nothing needs quoting, else double-quoted (pwsh/agent
panes single-quote a path holding `$` or a backtick; an UNSET shell is spelled for PowerShell,
the core's own Windows default, so a `$` path is not expanded). The image receipt strips one pair
of quotes before reading the extension. `isShellCommand` likewise normalises the session host's
image names (`pwsh.exe`, `C:\…\cmd.exe`), leading `-`, either separator, case, `.exe`, which is
why the session host's `paneCommand` walk (`process-tree.ts`) answers a shell name only when EVERY
process in the tree is a shell: claude's Bash/PowerShell tool or a `cmd /c npx` MCP launcher is a
shell LEAF under a live agent, and must not read as "the CLI let go of the pane". A path an
agent TUI wrapped under a **hanging indent** (Codex: 2 spaces, broken after a `/` or at a space)
is neither a soft nor a hard wrap, so it is offered as extra READINGS (`hangingReadings`, the
seam as nothing, and as one space), tried BEFORE the plain row's tokens by both the provider and
`linkAtCell`; existence decides, and a click on the indent is not a click on the path.

## Link menu

**Right-click on a link** opens a link menu (pure `terminal/link-menu.ts`; the listener is
`installLinkContextMenu`, beside the Cmd+click fallback and sharing its `linkAtCell` hit-test):
Open / Reveal in Explorer / Download / Copy path for a file, Open in browser / canvas browser /
Copy link for a URL. The gates are the Explorer's, reused not restated, Download only where
`downloadRoute` ≠ `none` (SSH project → scp to this machine, Server Edition → HTTP), never on a
desktop LOCAL project; the OS reveal only under `canUseLocalShell`. **The decision is made on the
right PRESS, not on `contextmenu`:** tmux 3.x binds `MouseDown3Pane` to its own `display-menu`, so
the press is what must be swallowed; a right-click OFF a link stays byte-identical (tmux menu,
agent TUI, node menu). A path-shaped token that turns out not to exist still gets a menu ("Not
found" + Copy path; "Couldn't check: <reason>" when its existence could not be checked), its
press was already swallowed, and a silent swallow reads as broken.
Downloads report in a `DownloadStrip` floated over the terminal, not in a drawer that may be
shut. The kanban card modal's right-click menu has URL rows only and no "Open in canvas
browser" (the node would land under the board). Its Cmd/Ctrl+click DOES follow file links, with
main's token model and lookup, but routed by the CARD's project, not the active one
(`lib/cardFileLinks.ts`, the Omni board opens cards from every project; any doubt turns file
links off): a file opens in `LocalFilePreviewModal` over the card (HTML copied into the agent-web
jail and shown under its strict CSP; source text in a browser tab, which has no webview), whose
"Open on canvas" is offered only for a card of the active project. On the canvas a LOCAL `.html`
opened by a link or that button renders as a WebNode (`fileViewerKind`'s `renderHtml`, carried as
`view: true` on `nodeterm:open-file`); Explorer, ⌘K and the files node still open it in the editor.

## Link hover

**Hovering a link says what a click opens** (`terminal/link-hover.ts`): the RESOLVED absolute
path, which of the two cwds held it, plus the gestures, `<abs> (⌘-click to open · ⇧⌘-click to
open with default app)` (Ctrl/Shift+Ctrl off-mac; a directory reads "reveal" / "open in
Finder|file manager"; a URL just `<url> (⌘-click to open)`, OSC 8 included, whose target the label
hides). It rides xterm's `ILink.hover`/`leave`, which fire in a tmux pane too, the linkifier
listens to `mousemove` on the screen element whatever the mouse-tracking mode; only CLICKS need
the capture fallback. One tooltip per xterm instance, INSIDE `term.element` (parks and dies with
the terminal, scales with the canvas zoom like the copy pill; positioned by dividing the rect by
the rendered/layout width ratio), `pointer-events: none` + `xterm-hover`, hidden by any press or
wheel. **Shift+Cmd/Ctrl+click opens with the OS default app** (`linkOpenIntent`, ONE routing rule
for the provider `activate` and `installLinkClickFallback`): `shell.openPath` behind
`canUseLocalShell`, the same gate as Reveal in Finder, so a directory opens in the OS file
manager. Everywhere that gate says no the click TOASTS its reason (`systemOpenRefusal`) and the
hint omits the gesture: an SSH project is refused rather than downloaded-then-opened (a click must
not silently copy a file or folder to this machine, and edits would land on a stale copy, the
link menu's Download is one right-click away), the Server Edition is refused rather than falling
back to the plain open (a modified gesture that quietly does something else is harder to learn;
the bridge's `shell.openPath` stays its documented inert stub), and a relay tab is refused (the
path is on the peer). Shift alone is never a link gesture (xterm's selection modifier); a
modified press released on another cell is a drag and is left alone, and a Shift+Cmd click that
extended an xterm selection does not open.

## Home-relative tokens

**Home-relative `~/x` tokens** (Claude Code prints its plan file as `~/.claude/plans/<name>.md`)
stay `~`-rooted all the way to the fs call and are expanded by the core that OWNS the filesystem
`expandHomePath` in `core/fs-handlers.ts` for desktop/Server Edition, the remote shell for
`sshFs`, because the renderer does not know that home. A `~` after a path character (`a~/x`)
or `~user/x` is not a home path and yields no link. The relay's jailed `fs.*` calls fs-ops
directly and does not expand, so a relay tab's `~` token fails closed (no link).

## Agent nodes

**Agent** (`createAgentNode(agentId, …)`), a terminal preset that runs an agent CLI as its
`initialCommand` (runs once on open via `transport.write`, then cleared), with `data.agentId`
set. Builtins (`claude`/`codex`/`gemini`) come from `AGENT_CONFIG` (clay color etc.).
Agent nodes get extra behavior **gated by the
agent's capabilities** (see [agent notifications and status](notifications-and-status.md)): a busy/working badge + unread dot +
completion notification + session-name chip (hook-capable agents), content search, and the
Claude-only **Branch conversation** action. Custom user-defined agents spawn + show
process/terminal-title status only.

## Overlays and wheel routing

**An overlay you lay over a live terminal steals its wheel, give it `pointer-events: none`.**
Wheel routing is a per-packet hit test on `closest('.nowheel')` (ours in `Canvas.tsx`, and React
Flow's own `panOnScroll` independently), so the element under the pointer decides, not the node.
`.term-node__xterm` carries the class and covers the whole body, so a banner drawn ON a running
terminal takes those pixels out of the terminal's wheel area and hands them to the canvas, the
user scrolls and the canvas slides instead. `.term-node__upload` and `.term-copy-pill` were
already `pointer-events: none` for this; `.term-node__stalecwd` was not (issue #767), and
`canvas/terminal-wheel-boundary.test.ts` now fails on the next one. An overlay that REPLACES a
dead view keeps the canvas wheel on purpose, there is nothing underneath to scroll. Same rule
one layer up: do not "fix" a routing question by moving `nowheel` outward, because a second
consumer reads the class and no per-element opt-out can reach it.
