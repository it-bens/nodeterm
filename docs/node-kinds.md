# Node kinds

Reference for the node kinds other than the terminal, browser, run and simulator nodes (which have their own docs: [terminal node](terminal-node.md), [browser node](browser-node.md), [run node](run-node.md), [simulator node](simulator-node.md)). Covers the persisted node shape, sticky, group, editor, diff, video, web, files, whiteboard, dino, trigger, the render-only subagent and loop cards, the removed chat node, Monaco and rendered Markdown, the node colour palette and media playback. See also [trigger node](trigger-node.md), [whiteboard node](whiteboard-node.md) and [node icons](node-icons.md).

## Node kinds at a glance

`NodeKind` (in `src/shared/types.ts`) is `terminal | sticky | group | editor | diff | video | web | browser | files | subagent | loop | dino | trigger | simulator | whiteboard`. `subagent` and `loop` are render-only (ephemeral hook-driven viz) and never persisted; everything else is persisted. Every node kind must be registered in `lib/reopenNode.ts` (either `UNRESTORABLE` or a `buildBase` case), or reopening it records a snapshot that can never be restored. Kanban cards derive only from an explicit list in `canvas/toKanbanSession.ts` (terminal, sticky and browser nodes); every other kind returns null there on purpose, because it has no session to co-attach or no text to edit and its value is spatial.

## Persisted node data and the trigger kind

The `trigger` kind is a first-class persisted kind. Its host-side machine is composed once in `core/trigger-service.ts` (`startTriggerService`) and booted identically by both shells. `core/trigger-scheduler.ts` (sweep-service shape, no catch-up for missed slots, cron via the dependency-free `@shared/cron` with the vixie dom/dow OR rule) decides
WHEN, and `core/trigger-delivery.ts` decides WHETHER, the `sendText` paste path, an agent target
only on a mirror-verified idle `done` (busy/blocked/unknown → the messaging `DeliveryQueue`, own
instance, flushed by the mirror's `done` edge via `onNodeStateChange`, with FULL flush-time
re-validation: a trigger disarmed or spec-edited while queued is dropped), a plain-terminal target
only into a SHELL pane and never queued, a dead target an honest `missed` and never a cold start.
Fire-time `TriggerArmStore.isArmed` re-ask everywhere; every rule test-pinned. Its spec (`CanvasNodeState.trigger`,
@shared/trigger) is git-shared CONTENT sanitized as hostile input on every load path
(`sanitizeNodeTriggers`), and the definition alone never fires, execution consent is the
machine-local, content-bound `core/trigger-arm-store.ts` (a spec that arrives or CHANGES via git
reads as disarmed until armed on this machine). A node's `data`
carries `title, color, group, tags, collapsed, expandedHeight, shell, cwd, text,
initialCommand, filePath, diffStaged`, `icon` (a user-chosen emoji, glyph or picture, see [node icons](node-icons.md)), `agentId` (which agent CLI a terminal node runs,
persisted), and `accountId` (which managed Claude account a terminal node runs under, resolved
at creation, changed ONLY by the explicit account-switch actions, persisted; see [managed accounts](managed-accounts.md)). `nodeStatesToFlow` defaults a
missing `kind` to `terminal` for backward compat and migrates the legacy `tags:['claude']` marker
to `data.agentId = 'claude'`. The SDK **chat node** was removed (2026-07); `nodeStatesToFlow` also
migrates a persisted `chat` node into a **sticky tombstone** in place, reading its legacy
`chatSessionId` to print a `claude --resume <id>` hint (a chat is an ordinary resumable Claude
session).


## Files a node owns, and default size

**Files a node owns are addressed by id, never by a path from node data.** `project.json` is
git-shared and hand-editable, so a path in it is attacker-controlled: a whiteboard's side file is
derived by the core from (project, node id) (`core/whiteboards.ts`), and the id must be a string
passing `isSafeNodeId`. A copy refers to its original by id (`whiteboardSource`), not by path.

**A new node kind also gets a default size the user can change.** Add it to `SizableNodeKind` /
`SIZABLE_NODE_KINDS` (`shared/default-node-size.ts`) and `BUILTIN_NODE_SIZES`, and place it with
`placeSized(kind, …)` in its factory. A literal `*_SIZE` passed to `placeNode` compiles fine and
quietly ignores Settings → Behavior → Default node size for that kind.

## Sticky

**sticky** (`StickyNode.tsx`), colored note, free text, collapsible. Has link handles:
connect a sticky to any terminal node to attach the note as context (see [context link](context-link.md)).

## Group

**group** (`GroupNode.tsx`), real React Flow parent/child frame, and frames **nest** (2026-08):
a group may contain other groups to any depth. `groupSelectedNodes` wraps objects that share ONE
container, frames included, creating the wrapper inside that container; a mixed-container set,
or an ancestor selected together with its own descendant, is **refused** rather than scrambled
(positions are only comparable within one container, and the descendant would be torn out of the
ancestor being wrapped). Box-selection routinely catches both, so structural actions normalize
the selection to its subtree roots first (`selectedRootIds`). `ungroupNodes` promotes a frame's
direct children into **its own parent** (not to the root, that would move them by the whole
ancestor offset); `reparentNode` moves a node OR a whole frame subtree, keeps its **root-space**
position fixed (`rootPosition`, not the old add-one-parent's-origin math) and refuses a cycle;
`addSelectionToGroup` adds a selection to an existing frame; `reorderGroupWithinParent` reorders
a frame among its siblings, carrying its subtree. `nodeStatesToFlow`/`groupsFirst` emit frames
**depth-first from the root**, a flat "groups first" sort is not enough once two groups compare
equal, and that persisted order is also the downgrade contract (a pre-nesting build's stable
sort leaves it alone, so a nested tree still hydrates parent-first and renders there). The order
has ONE definition (`groupsFirstBy`, `src/shared/node-order.ts`), and the shared op reducer
(`applyCanvasMutation`) re-sorts with it on an append or a `parentId` change, the same two points
the live React Flow apply does, because a governed Server Edition project's file is written from
the canvas authority's array, not from React Flow's.
**A frame that gains a child bigger than itself is re-fitted, ancestors included**
(`fitGroupToChildren` up the chain): a wrapper created at `(minX-28, minY-62)` relative to its
parent is routinely negative, and `extent:'parent'` would make React Flow clamp it into an
inverted range, snapping the frame hundreds of px away and dragging the whole wrapped subtree
with it. Visually: a dashed rounded frame in the group color with a floating label pill (color
dot + editable name) on the top border and ungroup/× top-right (on hover/selected). **The pill
is the frame's `dragHandle`** and the frame body is `pointer-events: none`, a frame is a
background container, not a giant drag target, so its body passes clicks to the pane and an
outer frame cannot swallow the clicks meant for a frame drawn inside it. The
`NodeResizer` line is hidden (`lineStyle` transparent) so it can't draw a sharp-cornered
box; the selection ring is a `box-shadow` instead, which follows the same `border-radius`.

## Editor

**editor** (`EditorNode.tsx`), Monaco code editor for a `filePath`; reads/writes via
`fs:read`/`fs:write`, auto-detects language from the path, ⌘S saves, dirty dot. A
**Preview / Edit** toggle (or ⌘M while hovered) renders the live content as markdown.
**Image files** (png/jpg/gif/webp/bmp/ico/svg/avif) skip Monaco and show an `<img>`
preview instead, read as base64 via `fs:read-binary` into a `data:` URL (CSP allows
`img-src data:`), on a checkerboard backdrop with the pixel dimensions in the header.

## Diff

**diff** (`DiffNode.tsx`), Monaco diff editor; `diffStaged` chooses HEAD↔index (staged)
vs index↔working (unstaged) via `git:show-file` + `fs:read`. Read-only.

## Video and audio

**video** (`VideoNode.tsx`), a video player; a local file is served over the `nt-media://`
protocol (allowlisted on mount via `media.allow`) with native controls; an SSH-project file
(`data.sshFs`) is first pulled into the local media cache over the project's ControlMaster
(`media.allowSsh`) then played the same way.

## Web

**web** (`WebNode.tsx`), an Electron `<webview>` (locked down, no `nodeintegration`) that loads
a live `data.url`, or serves local html at `data.filePath` over `nt-media://`.

## Files

**files** (`FilesNode.tsx`), a file-manager node: ONE directory listing (`data.cwd`, persisted),
pinned to the canvas beside the terminals working in it. Deliberately not a second Explorer: the
drawer is a single tree rooted at the project cwd that covers the canvas, so it gives you one
cursor and a lot of scrolling; several of these give you `src/renderer/nodes` next to the agent
editing it and another on `docs/`. Navigate in place (breadcrumbs collapse deep paths but every
crumb stays clickable), filter, create a file/folder, copy a path, reveal, and open a terminal in
the folder shown (a `nodeterm:open-terminal` event, the sibling of `nodeterm:open-file`).
- **It adds NO new IPC.** Everything runs on the existing `FsApi` (`list`/`mkdir`/`exists`/
  `write`), which is why it works on Desktop, the Server Edition, an SSH project and a relay tab
  on day one; `mkdir`/`exists` are genuinely LIVE on a relay tab (see the Explorer notes in [canvas panels](canvas-panels.md),
  that line used to claim otherwise), so "New folder…" works there too. **Rename/move/delete are
  the deliberate v1 gap**: each needs a new leaf in `core/fs-ops`, an IPC channel, preload, the
  ws-bridge, `main/ssh-fs` (remote quoting) and the relay host-service, plus confirm dialogs and
  dangerous-path guards. That is a separate change with separate risk, not a corner to cut inside
  this one.
- **Which filesystem** is `EditorNode`'s decision, read the same way: `data.sshFs` → the SSH
  project's host over the ControlMaster; otherwise the node's own SESSION api, which is the local
  core for a local project and the PEER's for a relay tab. Reading it off `useSession()` rather
  than `window.nodeTerminal` is the entire reason a relay tab browses the right machine.
- **Opening is delegated, never reimplemented**: a file dispatches `nodeterm:open-file`, so
  editor-vs-video-vs-image routing stays in Canvas's one `openFile` and this node never grows a
  second opinion about what a `.png` is. `fileOpenTarget` decides only canvas-vs-OS, and a REMOTE
  listing never reaches the OS branch, `shell.openPath` opens a path on THIS machine. **The OS
  branch is now also gated on being ABLE to open locally**, not just on remoteness: `shell.openPath`
  is a documented `noop` in the Server Edition (`bridge/stubs.ts:180-186`), and a browser tab's own
  session `source` is `'local'`, `SessionSource` declares `'server'` but nothing ever constructs
  it (`session/session.ts:8`), so `source` alone can never tell you you're in a browser, only
  `isBrowserRuntime()` can. Opening a `.zip`/`.dmg` was therefore a silent dead click; closed by
  `canUseLocalShell` (`lib/download.ts`), ONE predicate for every `shell.*` path action, with
  `canRevealLocally` kept as its older reveal-specific name so existing callers are untouched.
  Reveal was gated and openPath was not, and writing that rule twice is how they drifted.
- **Two state bugs, both fixed.** (a) A directory a removed worktree took with it used to render
  as "This folder is empty.", not "Could not read this folder", because `FsApi.list` is
  fail-open by contract (`core/fs-ops.listDir`/`SshFs.listDir` both end `catch { return [] }`, and
  the SSH IPC resolves `[]` even for a dead ControlMaster). `fs.exists` cannot disambiguate either:
  it's `stat`-based (true for a dir you can stat but not `readdir`), and on SSH its `false` can't
  separate "gone" from "the ControlMaster died", the same conflation `SshFs.readTextChecked`
  (`main/ssh-fs.ts:164-176`) refuses, on the rule "a failed read is never evidence of absence". Now
  disambiguated by probing the PARENT's listing instead (`classifyEmptyListing`, pure,
  `lib/filesNode.ts`, the idiom `SshProjectDialog.tsx:112-125` and `file-links.ts`'s
  `makeDirListingLookup` already use), which answers `missing` / `empty` / `unreachable` /
  `unknown`. The parent CONTAINS the directory we are standing in, so it cannot legitimately be
  childless: an empty parent listing means we could not see it, which is `unreachable` and gets
  its own sentence naming both possible causes. That case used to answer `unknown` and render as
  "This folder is empty." over a dead ControlMaster, the commonest cause of an empty remote
  listing and the most reassuring possible lie. **`unknown` is reserved for the paths whose parent CANNOT answer**, a non-`/`-absolute cwd
  (an SSH project's `remoteCwd` defaults to **`~`**, where `parentDir` is `/` and `/` has no entry
  named `~`, so an empty remote HOME reported a deletion), a `.git` cwd (both listing legs strip
  it on purpose), and `.`/`..` segments; a case-folded match counts as present, for the
  case-insensitive filesystems where `readdir` answers with the on-disk spelling. `missing` must
  be the conclusion we are SURE of. Nothing is published until the verdict is in, so a deleted
  folder never flashes "empty" on its way to the error.
  (b) Nothing reset the list on navigation, so "Loading…" was reachable only on the very first
  mount, every later directory change showed the PREVIOUS directory's rows. Fixed by storing the
  listing WITH the cwd it belongs to, so a cwd change IS the loading state by construction; a
  re-list after a create deliberately keeps its rows (same directory, re-read).
- The title tracks the folder only while `titleAuto` is unset, the same contract an agent node's
  session name uses, so navigating never overwrites a name the user typed.
- A files node inside a REMOVED worktree is displaced like an editor, not like a terminal
  (`displacedByWorktree`): it has no session to disturb, so it is caught by path wherever it sits.
  The patch is the pure `displacedFilesPatch` (`lib/filesNode.ts`), where `null` means LEAVE IT
  ALONE on the dead path, the parent probe then tells the truth, because
  `resetDisplacedCwd`'s fallback can be `undefined`, and writing that through would
  have cost the node the only thing it knows about itself. **The READ side is guarded too**: a
  files node with no `cwd` says so instead of falling back to `'/'`, `project.json` is
  git-shared, hand-editable input that nothing validates for this kind, so guarding only the
  writer left the silent root-browse reachable anyway. And `createFilesNode` places through
  `placeNode`, not `placeAt`, so snap-to-grid applies to its size as well as its position (React
  Flow resizes by adding a grid multiple to the START size, so an unsnapped box never lands on
  the grid later). The title
  rewrites alongside when `titleAuto` holds, the ONE cwd write that does not go through `navigate`.
- **"New terminal here" was broken on BOTH remote kinds, and not by this node's own doing.**
  `addTerminal` resolves the project from `activeProjectId` and `createTerminalNode` does
  `cwd: ssh ? ssh.remoteCwd : cwd` (`state/workspace.ts`), on an SSH project the folder on screen
  was silently DISCARDED and the terminal opened at the project root, a pre-existing hole in
  `addTerminal`'s `cwdOverride` contract affecting every caller, not just this one. Fixed by routing the call
  through the EXISTING `nodeSshFor`, which already carries this exact reasoning ("passing the
  project's ssh unchanged silently REPLACES the caller's cwd") and which `addTerminal` simply
  never used; it is handed `cwdOverride` and never the resolved `cwd`, because an SSH project can
  still carry a local `project.cwd` that `scmCwd` falls back to, and promoting that would root
  remote terminals at a path from the wrong machine. On a RELAY tab there is no `ssh` to
  rebind, so the row used to spawn a plain LOCAL terminal at the peer's remote path; it is now
  withheld there instead, spawning onto a peer's core is a real feature, not a one-liner.
- Creation needs a project directory (`hasCwd`), and **the row degrades EXPLICITLY rather than
  vanishing**, disabled, carrying `FILES_NO_CWD_HINT` (an alias of `NEW_FILE_NO_CWD_HINT`), on
  the pane menu, the Dock and the sidebar "+". That is the rule #621 established for "New file…"
  and the SSH worktree row: a cwd-less project is a supported, persisted canvas, so a row that
  simply disappears takes its own reason with it while the fix ("Set folder…") sits one menu
  away. **⌘K is the deliberate exception** and hides the entry, exactly as main's "New file…"
  does, a disabled palette entry surfaces as a search result that does nothing, the same reason
  `sshAccountsHint` is omitted there. Inside a group frame it inherits a bound **worktree's** cwd
  via `cwdForNewNodeIn`, so a frame per branch also means a file tree per branch.
- **A node kind not registered in `lib/reopenNode.ts` is a trap, and this one fell in it.** Every
  kind must sit in exactly one of `UNRESTORABLE` or a `buildBase` `case`, `files` was in neither,
  so ⇧⌘T recorded a snapshot (and a persisted `closedSessions` twin) that `buildBase`'s
  `default: return null` could never restore: a dead, clickable entry that compiles, typechecks and
  passes every test. `files` now has a `case` (it IS restorable, its whole state is the directory
  it shows), unlike `trigger`, excluded for the same missing-case reason
  (`reopenNode.ts:58-60`, the comment that made this findable). Two kinds have fallen in this trap
  now; treat registration as a checklist item for the next one.
- **Kanban: deliberately not a card.** `canvas/toKanbanSession.ts` maps `browser`, `sticky` and
  `terminal` and returns `null` for everything else, cards do not "derive from terminal nodes",
  they derive from that explicit list. A files node has no session to co-attach and no text to
  edit; its value is spatial adjacency to the terminals working in the directory, which a column
  layout would discard.
- `folderTitle` lives in `lib/explorerCreate.ts` (a zero-import leaf), not `lib/filesNode.ts`:
  `filesNode.ts` imports `isVideoFile` FROM `state/workspace`, so importing back would close a
  cycle. `filesNode.ts` re-exports it, so call sites are unchanged.
- **The `/`-separator assumption is a KNOWN gap, shared with `explorerCreate`** (the Explorer
  drawer and canvas "New file…" already run on the same helpers), `C:\x\y` reads as one segment.
  To be closed in ONE place for both, using the core-owns-the-dialect rule `terminal/file-links.ts`
  already implements. The TRAVERSAL half is closed: `newEntryPath` splits its `..` check on
  `[\\/]` and refuses a Windows-absolute name, because a guard that reads `\` as ordinary text is
  wrong about the machine that resolves the path. The construction half deliberately is not: on
  POSIX a backslash is legal filename text, so the join stays `/` and `weird\name.txt` is still
  one file. Guard on both dialects, construct in one.
- **Mobile**: N/A, *nodeterm mobile* attaches to tmux sessions over the transport protocol and
  has no canvas or file-browsing concept; adding one means extending that protocol.

## Whiteboard

**whiteboard** (`WhiteboardNode.tsx`, `whiteboard/WhiteboardEditor.tsx`), an Excalidraw board;
full write-up in `docs/whiteboard-node.md`. The invariants: (1) editing is ONE live editor
portaled into `.canvas-root` over the node's box, never inside the scaled node (pointer drift,
blur, and `.flow-wrap`'s wheel capture); (2) the scene is a side file the core addresses by
(project, node id), `core/whiteboards.ts`, and NEVER by a path from node data (project.json is
git-shared, hostile input); node data holds only `whiteboardSource` (a copy's original, by id)
and `boardRev` (once per session); (3) the overlay is `data-nt-keyboard-owner`, which every
typing-aware key site honours via `isKeyboardOwnerTarget`; (4) every save is a COMPARE-AND-SET:
it sends the revision its editor last read or wrote (`rev`, `dev:ino:size:mtimeNs` of the scene
file, writers publish by rename, so it changes on every write; `null` = no board of its own yet)
and the core refuses it when the file on disk has moved on, so a board open in two places (two
tabs, a desktop and a browser on one folder, two hosted-team Editors, a `git pull`) is never
silently replaced by the later writer, the editor says "Changed elsewhere" instead. Saves of one
board are serialized within a core; two CORES on one folder can still interleave between the
check and the rename. The editor's own "has anything changed" test compares with what IT last
saved, not with disk. The unmount flush writes a snapshot kept from `onChange`, Excalidraw's api
is empty by cleanup time, and reading it there once erased a board; (5) the project folder is
bound by its REAL path, a `.nodeterm` or `.nodeterm/whiteboards` symlink leading out of the
project is not followed (the board falls back to the app folder); (6) finishing (Esc, a press
outside) waits for a save that SUCCEEDED, a failed or refused one keeps the editor open with
"Keep mine" / "Discard my changes" in the header, since after the exit there is no flush left to
retry, and a hosted-team Viewer opens the board view-only. Excalidraw loads lazily; fonts
come from `scripts/excalidraw-assets-plugin.ts`; mermaid is aliased to a stub.

## Dino

**dino** (`DinoNode.tsx`), a small self-contained T-Rex-style runner on a canvas (no PTY);
high score persists via `data.highScore`.

## Trigger

**trigger** (`TriggerNode.tsx`), a canvas-owned schedule (cron / interval / once) that
delivers a payload into a connected terminal/agent node when due (issue #493, the inverse of
the ephemeral loop/cron cards, which visualize AGENT-initiated recurrence). The card shows the
schedule + next-run countdown, the target (a derived, never-persisted edge, the
pending-launch dep edge is NO longer one: since the 2026-09-02 edge model it is a persisted rope,
`ctrl-after-<dep>-<node>`, whose dashed ⏳ LOOK is what is derived), the payload, an honest
ARMED/DISARMED/CHANGED/SET-UP chip with the
"definitions travel with the repo, consent never does" narrative, Run-now, and the last runs
(fired / delivered-late / queued / missed / failed / expired). Arming passes a ConfirmDialog
showing the exact schedule+payload+target being consented to; all decisions are the pure,
tested `lib/triggerCard.ts`, all state is host-side over `window.nodeTerminal.triggers`
(arm/disarm/status/runNow, `startTriggerService` registers the handlers in BOTH shells;
`runNow` deliberately takes no spec: a caller chooses WHEN, never WHAT). The relay stub
refuses and the card says triggers are managed on the host. Mobile: N/A (no canvas).

## Subagent and loop cards

**subagent** / **loop** (`SubagentNode.tsx` / `LoopNode.tsx`), render-only, hook-driven viz
nodes, **never persisted**. `subagent` visualizes a subagent the Claude session spawned (type +
task + live timer, expand for its live transcript, subagents have no PTY); `loop` shows a
loop/schedule/cron kind + task + per-iteration summaries, Play re-issues the task into the parent
terminal's tmux session.

## Chat node (removed)

**chat**, **REMOVED 2026-07.** The SDK-driven Claude chat node (`ChatNode.tsx`, `main/chat-driver.ts`,
the `@anthropic-ai/claude-agent-sdk` dependency, and the whole chat-events/chatSessions stack) is
gone, dropping the bundled SDK also removed a ~240 MB native binary per platform. A persisted `chat`
node is migrated by `nodeStatesToFlow` into a **sticky tombstone** in place, carrying a
`claude --resume <chatSessionId>` hint so the conversation continues in any terminal (a chat was an
ordinary resumable Claude session). `CHAT_CAPABLE` / `canChat` survive but now gate **only** the
⌘M **ChatPanel** transcript view on a Claude *terminal* node (see the markdown view in [terminal node](terminal-node.md)),
not any SDK chat node.

## Monaco and Markdown rendering

Monaco is wired in `renderer/editor/monaco-setup.ts` (language workers bundled via Vite
`?worker`, no CDN; CSP `worker-src` allows them). Markdown rendering is shared in
`renderer/lib/markdown.ts` (`marked` + DOMPurify sanitize). Terminal OUTPUT (the ⌘M output view)
goes through `renderer/lib/terminalOutputMarkdown.ts` instead: a private `Marked` instance with
`breaks` on (output is line-oriented, without it `ls -l` joined into one paragraph) that renders
raw-HTML tokens as escaped TEXT (a program's `<stdin>` is not markup; parsed as HTML, DOMPurify
stripped it) and trims capture-pane's trailing padding. The escape is on the `html` token, never a
global pre-escape of `<`, which would double-escape code spans/fences.

### TS/JS worker reaping

**The TS/JS worker is reaped, not idled** (`renderer/editor/tsWorkerReaper.ts`). Monaco 0.57 stops
the css/html/json workers after 2 min idle but never the TypeScript one (separate TS and JS
managers can pin two compiler isolates; their size has not been measured here). monaco-setup stops
it with a no-op `setCompilerOptions` on `monaco.typescript.*Defaults` (`onDidChange` →
`_stopWorker`; the next request restarts it lazily) 2 min after the last TS/JS model is disposed or
changes language. This only works if every node **disposes its models on unmount**: one leaked
`.ts` model pins the worker for the session.

### Rendered-Markdown font

**The rendered-Markdown font is a setting** (`markdownFontFamily` / `markdownFontSize`, Settings →
Appearance; `@shared/markdown-font`). App.tsx publishes it as `--md-font-family` / `--md-font-size`
on <html>, plus `--md-code-font-family` = the TERMINAL font family; styles.css keeps the defaults on
`:root`. Every rule that sizes rendered Markdown reads the tokens, `.term-md__content` (⌘M output
view, editor preview, the kanban file preview), `.term-chat__msg` and the ⌘M composer input,
headings are `em`, and code is `0.92em` of the body (relative, so it stays in proportion).
`styles.markdown-font.test.ts` pins this. Sticky notes keep their own font on purpose.

### Rendered markdown containers and spinners

**Rendered markdown goes inside a listed container.** A link in `renderMarkdown` output keeps its
href as written, and a relative one used to navigate the whole app window away (the canvas was
gone until a reload). One delegated handler (`renderer/lib/markdownLinks.ts`) intercepts clicks
inside `RENDERED_MARKDOWN_CONTAINERS`; a new surface that injects markdown HTML must use one of
those classes or join the list, `markdownLinks.test.ts` fails otherwise.
- **A loading indicator is `components/Spinner`, never a local spinner.** `.nt-spinner` is the one
ring in `styles.css`, and it freezes under `prefers-reduced-motion`; a local copy is how a
spinner kept rotating for users who asked for no motion, and how two rings swapped mid-load. Put
`role="status"` on the row holding the text, not on a second spinner beside it.


## Node colors (one palette, two sections)

`src/shared/node-colors.ts` is the palette AND the control boundary, the picker's list and the
allowlist `color --color C` validates against are the same array, deliberately, so the two can
never disagree about what a legal colour is.

It has two sections. The seven macOS **system** colours are the list it always was. The **agent**
section is the brand colour of every builtin in `AGENT_CONFIG` plus `FALLBACK_AGENT_COLOR`, and it
exists because those colours were *already on the canvas*: `createAgentNode` paints a new node from
`AGENT_CONFIG` (the phone's `appendProjectNode` does the same), so a Claude node is born `#d97757`
which the picker could not offer back once the user changed it, and which
`nodeterm color --color '#d97757'` refused with `node-color-invalid`, the app rejecting its own
colour. MEASURED against the live hook server before the change; both faces are one gap.

- **The agent section is DERIVED from `AGENT_CONFIG`, never re-typed.** Adding a builtin agent
  extends the palette by construction. `node-colors.test.ts` pins it three ways, every builtin's
  colour is in `NODE_COLORS`, every agent swatch's label is that agent's label, and `node-colors.ts`
  contains no agent hex as a literal on a non-comment line. A second copy of a colour table is the
  drift this file warns about everywhere else, and it has already happened once (the mobile note follows).
- **`SYSTEM_NODE_COLORS` is the subset for surfaces where the value is drawn as TEXT or as an
  opaque fill**, the app accent (`--accent` sits under hardcoded `#fff`), a project colour (the
  active tab's label is `color: p.color`) and a kanban column colour (`ColumnPill` draws the title
  in the raw hex at 10 px). It is also the **auto-assign rotation** for new frames, spawned teams
  and fresh columns: rotating a frame onto Claude's orange says "this frame is Claude's" and means
  nothing of the kind. The reason those surfaces differ is a measurement, not taste, on the dark
  theme white on gemini `#4285f4` is ~3.6:1 and on grok `#64748b` ~4.0:1, both under the 4.5:1 floor
  for the 10.5 px badge, and grok grey as tab text is ~2.6:1. Everywhere the colour is a dot, a
  border or a 6–20 % wash (node headers, sticky, files, group frames, the account default colour)
  offers the FULL palette.
- **`resolveNodeColor` runs BEFORE the allowlist, and only its output is ever persisted.** It takes
  a palette name (`blue`, `teal`, `claude`, `github copilot`) or the hex in any case, and yields a
  canonical value or `undefined`. Names are accepted because the refusal they earned was measured
  and expensive: seven opaque hexes taught nobody which one was teal, so a caller's next guess was
  another name and another refusal. The boundary is unchanged by this, a name can only ever
  resolve to a value the allowlist already held, and a name is never stored. The refusal now prints
  `name #hex` per swatch (`nodeColorChoices`), and the agent-facing skill text is generated from the
  same function per the canvas-control sync rule.
  `open-project --color` takes the same treatment against the **system** resolver; it used to reach
  `registerProject` with no validation at all.
- **One component draws every picker** (`components/NodeColorSwatches.tsx`), because the palette now
  has structure, headings and per-swatch names, and six copies of `NODE_COLORS.map(...)` are six
  places to forget the heading. `NodeColorSwatches.guard.test.ts` fails on a `.color-popover` this
  component does not own and on any renderer file that maps the palette into its own swatch row.
  The name is the feature: an agent brand colour is unidentifiable as a bare circle.
- **Mobile keeps its OWN list and is not updated here.** *nodeterm mobile* (`nodeterm-ios`,
  separate private repo) hand-copies the agent colours in `NewSession.swift`, stamped
  *"last verified 2026-07-17"*, to colour a session it creates; it has no node-colour picker at all,
  so the palette change reaches it as nothing to do. That mirror is the concrete precedent for why
  the desktop half is derived rather than typed, and it means a future brand-colour change owes an
  iOS follow-up (@eneskirca), not a desktop one.



When adding a surface that lets someone choose a node colour, render `<NodeColorSwatches>` rather than mapping the array yourself (a guard test fails on a hand-rolled swatch row). If the value will be drawn as text or as an opaque fill under white, take `SYSTEM_NODE_COLOR_SWATCHES` instead, with the contrast reason in a comment.

## Media playback lifetime (#680)

Audio routes through `isMediaFile` to the existing persisted `video` kind and native audio controls.
Legacy audio editor nodes migrate on load. Desktop uses the existing local/SSH allowlist; Server
and relay retain their explicit media transport degradation. Native mobile does not consume these
React nodes; its own player/IME behavior requires a separate device check.

Remote cache entries requested in this run are retained until exit, including cache hits; pruning
waits are coordinated with a concurrent cache reader. The 20-entry cap is soft for retained files,
so disk use may grow in a long run; prior-run entries are eligible again after restart. `mediaRange`
handles suffix ranges and rejects unsatisfiable ones without weakening the serve-time path jail.

Media URLs live for the app run. Remote cache pruning must keep files already handed to
players; the cache cap is soft until restart. Use the existing `video` node for audio too.