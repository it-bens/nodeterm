# Kanban board

The kanban board is a full-page view over the canvas, per project, plus an optional all-projects swimlane view (Omni). This doc covers the board's data model, card sources, card modal, comments and board log, multi-select, ordering, saved views, keyboard, team progress, the phone's board verbs and the shared node-action builder. Code lives in `src/renderer/components/kanban/` (`KanbanView.tsx`, `GlobalKanbanView.tsx`, `CardModal.tsx`), `src/renderer/lib/kanban.ts` and `src/shared/kanban-*.ts`. GitHub-backed cards are covered in [github-sync](github-sync.md), [github-pull-status](github-pull-status.md) and [board-dispatch](board-dispatch.md); the earlier GitHub issues design note is [github-issues-kanban](github-issues-kanban.md).

## Canvas and board are two views of the same nodes

A feature added to a canvas node (a header action, a badge, a menu item, file drop, dictation) is not done until the board's card and card modal have been considered too, in the same change. The board already mirrors most of the node surface: the card modal co-attaches the same tmux session (`ModalTerminal`), carries the node's actions (search, dictate, AI-name, comments), accepts file drops (`terminal/file-drop.ts`), renders browser webviews (`BrowserSurface`) and supports right-click actions and `+ New`.

**Node actions live in one builder**, `renderer/lib/nodeActionItems.tsx`, never inline in `Canvas.tsx`. The canvas node menu, the sessions-sidebar row menu and both boards' card menus (through `components/kanban/cardMenu.tsx`) build from it. A card takes only `BOARD_NODE_ACTION_IDS`; a row that acts on canvas position goes in `BOARD_SPATIAL_ROW_IDS` instead. The card modal's ⋯ menu takes `BOARD_MODAL_ACTION_IDS` through its `sessionActions` prop and closes the modal before any action runs, because restart-and-shell, switch model, account switch and pause-and-end recycle the tmux session its terminal is attached to. A card of a project the canvas does not hold shows these rows disabled with "This terminal is not attached right now."; a card with no session row (a plain shell, a non-resumable agent) gets no ⋯ button at all. A node the canvas holds whose terminal was released while off screen is a different case (`NodeActionCtx.onCanvas` true, `attached` false): Refresh stays enabled, since its respawn is what revives the node, and the session rows say the terminal was released (`RELEASED_HINT`).

The board acts on nodes of projects that are not on the canvas (the Omni board), so writes go through Canvas's `nodeWritesFor(projectId)` (built on `createNodeWriteRouter`, `lib/nodeWriteRouter.ts`), never a live-canvas callback, and failures report with a `nodeterm:toast`. `lib/nodeActionItems.guard.test.ts` fails on a second copy of the rows.

## Opening the board

The toggle is a Trello-style icon on the active project tab (`.tab__board-toggle`, after the name, before the caret) plus ⌘⇧B and ⌘K. The view belongs to the project, which is why the icon sits on its tab (the tab-strip end and the controls cluster were both tried and rejected in use). The board is a per-project overlay over the canvas.

The canvas stays MOUNTED under the opaque overlay: agent-status listeners live in `Canvas.tsx`, and `display: none` would resize every terminal to 0x0 and send tmux a SIGWINCH. Canvas-only shortcuts (undo, ⌘T/⌘⇧C, Delete) early-return via `isKanbanOpen`.

Z-order contract: overlay 25 < `.controls-cluster` 26 (Explorer, Source Control and Settings stay clickable on the board) < `.top-banners` 27 (a mandatory-update card must not hide behind the board) < tab bar 30.

The board opens with a title strip (`.kanban-header`: project dot and name) whose height clears the floating controls-cluster icons, so columns never sit under them.

## Card sources: a registry, not a branch per call site

The board is dual-source. SESSION cards are the project's session nodes (React Flow type `terminal`), derived live from the canvas nodes (title, color, kind, agentId), with RUNNING / NEEDS YOU badges and an unread dot from the default `agentStatus` store (click returns to the canvas and runs `focusNodeById`). GITHUB cards are the repo's issues and pull requests (`GitHubIssueCardView` via `state/githubIssues.ts`, opened through `GitHubIssueSummaryModal`; a column move that closes or reopens an issue confirms first). A source filter (`KanbanSourceFilter`: All / Issues / Pull requests / Sessions) and a transient per-board label filter narrow what shows.

Where a card comes from is declared once in `renderer/lib/kanbanSources.ts`, the same membership-plus-one-leaf discipline `AGENT_CONFIG` uses. Each entry declares:
- its filter `label`;
- its `placement`: `assignment` (the board's own persisted assignments, reorderable within a column) or `provider` (the provider reports the column, the board persists nothing and a move is the provider's write);
- its in-column `lane` order;
- whether it is `configured` for a given board;
- whether it is `readOnly` (the board never writes it: no drag, no move control).

Two orders live there on purpose: declaration order is the source filter's button order (All, GitHub, Sessions), `lane` is the in-column stacking order (sessions above issues). They genuinely differ, and pinning both stops either being re-spelled elsewhere. `KanbanColumn` takes ONE `lanes` prop (`{sourceId, cards, footer?, count}`) instead of a `cards` prop plus eight `github*` props, places lanes via `byLane` and names no source; the board builds each source's leaf, and the drag union branches on `placement` (`isProviderDrag`), not on the string `'github'`. A lane's `count` is passed rather than derived from `cards.length` because a provider reports a server-side total larger than the page fetched so far. Writing `=== 'github'` outside the registry means the registry is missing a field.

What deliberately did not move into the registry: the virtual Ungrouped column (board semantics, not a source's concern) and `validKanban`, which stays the single shape gate on every load path. A registry entry must never grow its own parallel validation.

## Data model and tiers

Board data is `project.kanban` (`{columns, assignments: [{nodeId, columnId}]}`, order = array order) in `.nodeterm/project.json`. It is git-shared, rides rev, mirror and watcher, and is absent until the first edit (`defaultKanban` seeds To Do / In Progress / Done). View choice is personal (`state/viewMode.ts`, localStorage `nodeterm.projectView`). Pure transforms live in `renderer/lib/kanban.ts`.

A board feature must pick one of three tiers:
- A board FACT is shared content in `project.kanban`: optional, sanitized, harmless to an older build.
- A DISPLAY preference is per-user localStorage (`state/kanbanDisplay.ts`, `nodeterm.kanbanDisplay`, per project).
- A filter on LIVE agent state is component state only and never persisted.

The virtual Ungrouped column (never persisted, undeletable, unrenamable, always first) holds every session with no assignment or a dangling one, in canvas order, so the board never opens empty. Assignment is board metadata only: drags never move canvas nodes or change groups. Assignments of dead nodes prune lazily on each board change (`pruneAssignments`). Column delete needs no confirm (cards return to Ungrouped; there is no last-column rule because Ungrouped remains). Per-column "+ New session" menus create agents, terminals and sticky nodes assigned to the column; the assignment is written UN-pruned because the fresh node is not in the derived list yet.

### `sanitizeKanban` is the shape rule

`sanitizeKanban` (`core/workspace-files.ts`) runs on all three seams: `fileToProject`, the store's inline-project branch (cwd-less projects bypass `fileToProject`) and `projectToFile` on the way out (the two-seam rule `sanitizeLayouts` follows). It is `validKanban` plus per-entry repairs, never inventions:
- a column that is not an object with string `id` and `title` is dropped (React cannot render an object title, and a render throw boot-loops the app because the view choice persists);
- a non-string `category` is dropped;
- a malformed assignment is dropped;
- a card's `assignees` that is not a list is dropped (entries without a string name and colour are filtered);
- every other field round-trips;
- a clean board comes back BY IDENTITY, so a well-formed file is never rewritten.

A v1 `{columns, cards}` board or a hand-mangled one falls to the fresh default instead of crashing the render. Readers do not trust the load path alone: `cardAssignees` (`@shared/kanban-labels`) is the one reader of `meta[].assignees` (board, card, member filter, log diff and handoff pings all use it), because `assignees: 5` iterated raw threw during render and inside the `assign` verb.

`bridges` and `ropes` are admitted through `sanitizeLinks` (`core/workspace-files.ts`) on the same seams (`fileToProject`, `projectToFile`, the inline-project branch, the legacy v2 path of the store) and on `persistedCanvases`, which reads the raw index entry and last-written file for the context-link map (`buildBackgroundLinkMaps` iterates every bridge). They are git-shared, hand-editable input, and the canvas's rope restore threw on one `null` entry at project load. A non-list is dropped, an entry without non-empty string `id`/`source`/`target` is dropped, and a clean list comes back by identity.

**Do not add a field inside `ProjectKanban.meta[]` entries.** Every card-meta setter (assignees, due, priority, labels, the phone's label verb) rebuilds the entry from a fixed field list and silently drops anything else. Board-level fields survive every transform; `pullLinks` is one.

### Lifecycle category

`KanbanColumn.category?: unstarted|started|done|closed` (`@shared/kanban-category`). Every reader goes through `columnCategory`: an unknown STRING reads as absent but is KEPT in the file (dropping it would erase a newer build's value on an older teammate's save); a non-string never reaches a comparison. The default board carries unstarted/started/done on every seeding surface (`defaultBoardColumns`, shared by `defaultKanban` and the relay's `ensureProjectBoard` and label seeding). Category drives:
- the header progress (`boardProgress`: live cards in done and closed columns over EVERY live card, Ungrouped included; null when no column is done or closed, since a number over an undefined "complete" would be invented);
- hiding `closed` columns behind a per-user toggle (default hidden);
- the GitHub completion column's default (`defaultCompletionColumnId`: first done, else first closed, else the last column, which is the pre-category default, so an uncategorized board is unchanged).

A category change on a column that holds cards is never silent. It is a claim about every card in the column (they start counting as finished, or leave view when it becomes closed), so `categoryChangeImpact` gates a `ConfirmDialog` naming the count and the consequence; an empty column changes at once. Confirm rather than refuse: refusing would only make the user empty the column first, which is friction, not safety. It is set from the column's ⋯ menu or header right-click on the per-project board; Omni shows no column menu.

### Placement and rank strings

An unanchored card move lands at the TOP (`assignNode`): no `before`, or a `before` naming a card outside the destination column. An agent's `assign` into a long Done column used to append at the bottom and read as "disappeared". A positional drop still says where it landed: below the last card or on the column body passes `AT_COLUMN_END` explicitly (both board views). The relay's `projects.setCardColumn` follows the same rule, and the `assign` help in both agent bodies says so (`canvas-control-core.test.ts` pins it).

Order within a column is `KanbanAssignment.rank?` (`@shared/kanban-rank`, `@shared/kanban-order`).
- An entry without a valid rank (every pre-rank board, an older build's move since its `assignNode` rebuilds the moved entry without the field, a hand edit) sits right after the entry before it in the ARRAY; ties keep array order.
- Keys are base-62 fractional-index strings with an integer head, so the default top insert is a decrement: a thousand top inserts stay four characters (a digits-only midpoint scheme grows a character every few).
- A move never throws. About 1.3k inserts into ONE gap grow a key past `RANK_MAX_LENGTH` (256); only then is the destination column re-keyed evenly as one contiguous block (`rebalanceColumn`), the one rebalance this scheme does.
- A rank STRING readers cannot use is kept in the file (like an unknown category) and re-keyed by the next write into that column; a non-string is dropped.
- A card assigned twice (a clean git merge can do that) keeps its FIRST assignment everywhere: `sanitizeKanban`, `columnOrder` and `placeAssignment` (which removes every copy of the moved card).

**Every write goes through `placeAssignment`** (`@shared/kanban-order`; renderer `assignNode` AND the relay's `setProjectCardColumn`). It also keeps the assignments array in rank order so a build that ignores `rank` shows the same column. The two goals ("a move is a one-line diff" and "keep the array in rank order") pull against each other; the resolution is to move as little as the second allows. A cross-column move whose array slot already sits between its new neighbours changes ONE entry in place (two lines: `columnId` and `rank`); otherwise its block moves next to its successor; a reorder WITHIN a column always moves a block (the array must change for an old build to see it). A destination column is repaired first when it must be: missing, invalid or colliding ranks are re-keyed in the order the column was already showing (only those entries change; a board's first write into an unranked column ranks it once), and an array that disagrees is re-slotted within the column's own positions. Never splice the assignments array by hand.

MEASURED with `git merge-file` (`core/kanban-rank-merge.test.ts`): two ADJACENT cards filed into Done concurrently conflict under array-only placement and merge cleanly with ranks; two NON-adjacent ones merged cleanly either way, so claim no more than that. The file's own `rev`/`savedAt` header still conflicts on any concurrent save.

### Labels, status chips, saved views

Labels are a per-project palette (`ProjectKanban` labels, edited inline through the Notion-style `LabelPicker`: create, assign, rename, recolor, delete via the pure `lib/kanban.ts` transforms) plus each GitHub issue's own labels, both filterable.

Status chips (Running / Needs you / Unread, `lib/kanbanStatusChips.ts`) read the store through a derived primitive signature (`statusChipSig`), never `byId` (the `armedDepSig` rule), and are never persisted: a filter on second-by-second state that survived a restart shows a wrong board. The card badge and the chips share ONE rule, `cardBadge`, so a chip cannot select cards whose badge says something else. They narrow session cards only (like local labels), AND with the label filter and OR within themselves.

Saved views (`kanban.views: [{id, name, query}]`, `@shared/kanban-views`) are shared content. A query carries source, labels, members and columns (the member and column filters are board filters of their own). `viewQuery` is the ONLY builder and reads only those four, so the status chips can never enter a view. The ACTIVE view and "show closed" are per user (`kanbanDisplay`) and restored on entering the board; a view a teammate deleted is ignored. Deleting a view confirms, because it goes for everyone. `sanitizeViews` runs inside `sanitizeKanban`, keeps query fields it does not know, and `KANBAN_VIEW_SOURCES` is pinned to the renderer's source registry (the shared sanitizer cannot import it).

## Cards

Cards collapse and expand on single click (transient state). The expanded row reuses `ContextMeter` (model and % pill, as in the node header), the session chip and an ↗ open-on-canvas button; double-click opens the node directly.

**The card does not repeat what its place says** (`lib/cardRedundancy.ts`). The session-name chip is hidden when it is the card's title (agent titles auto-track it), the same rule the canvas node header always had, now one function for both. A past due date in a done or closed column keeps its date but drops the overdue alarm. The card MODAL keeps both facts: its header names a session that differs from the title, and its Due strip still says "Overdue". Audited and not hidden: the card never names its column or its project, and no badge restates a column category, because RUNNING / NEEDS YOU describe the agent's turn right now, which is what an idle card in In Progress needs distinguished.

An assigned session node shows its column as a half-pill flush on the node's top edge (`components/kanban/ColumnPill.tsx`, `columnForNode` in `lib/kanban`). It is rendered as a SIBLING of the node root because the roots are `overflow: hidden`; it is hidden for Ungrouped and dangling assignments, and a click opens the board.

### Multi-select (both boards)

⌘/Ctrl-click toggles a session card, Shift-click extends from the last picked card in BOARD order, and while anything is selected a plain click keeps picking instead of opening the modal (Esc or the bar's ✕ leave; the card menu's Select row is the way in without a modifier). The floating `CardSelectionBar` and the right-click menu of a card inside a multi-card selection (`buildBulkCardMenuItems`) act on all of them: **Move to** (`assignNodes`, which lands the group together and in board order at every anchor; a loop of `assignNode` would reverse it at the top) and **Delete** (ONE canvas confirm: `onDeleteNodes`, or `nodeterm:global-delete` with `nodeIds` from an Omni lane). Dragging a selected card carries the selection.

Load-bearing: the selection is pruned against the cards ON SCREEN and callers read the pruned value in the same render (`useCardSelection`), so a bulk delete can never reach a card that a filter, a collapsed lane or a hidden closed column took out of sight. GitHub cards are never selectable (their column is the provider's). On the Omni board a selection lives in ONE lane (`selectionOwner`, since two projects share no columns). Rules are pure in `lib/cardSelection.ts`. No keyboard chord yet (no `board.*` registry command selects or deletes). Scope limits: no agent-driven card movement from this feature, no board undo.

## Card modal

A card's ↗ or double-click opens the card modal (`components/kanban/CardModal.tsx`): body portal on the dialog stack, scrim z 55, scrim and Esc close (Esc in CAPTURE phase; an Esc during a header rename only cancels the edit). It runs under the CARD project's session (`ProjectSessionScope`; the boards themselves sit under App's local provider).

Terminal cards get a LIVE second view of the tmux session (`ModalTerminal.tsx`). The pty subscriber ledger is keyed by the composite `(ClientId, viewerId ?? PRIMARY)` (`core/pty-manager.ts`, viewer identity); `viewerId` is an optional TRAILING argument through preload, ws-bridge and `LocalTransport`, absent = bit-for-bit legacy, and a client's per-connection socket pause survives a single view's departure. The modal viewer seed-paints from the joiner screen (`toXtermText` transforms, since raw capture-pane output staircases), handles fresh-cold start via scrollback snapshot and a hint (agent auto-resume stays canvas-only), deliberately has no park, WebGL, hover or flow control, and kills ONLY its own viewer on close. Sticky cards edit their text in the modal (live both ways).

The modal header carries the terminal node's actions: search (`useTerminalSearch` and `FindBar` on the modal xterm), dictate (the same `nodeterm:dictate` event; the `.dictation` overlay is z 60, above the modal scrim), ✦ `pty.generateName` through the modal rename funnel, and the pause chips DROPPED, PAUSED and SLEEPING (Eco, with the refused-wake sentence), each clicking through the same `wakeHibernatedNode` trigger as the canvas chip. The modal overlay sits at z 5 in the pane, BELOW the sheet's resize handles (z 6/7), pinned in `styles.kanban.test.ts`.

**The ⌘M view in the modal** is a header toggle (`IconMarkdown`) plus the chord, which lays the SAME face the canvas node shows over the live viewer: `ChatPanel` when `canChat(created agent)` and the session id is known, else `TerminalMarkdownView`. Three rules:
1. The state is MODAL-LOCAL and per opening, never `data.mdMode` (that would flip the canvas node under the board too).
2. The viewer stays MOUNTED underneath (covered, not swapped, so its co-attach never detaches and re-attaches) and gets the same focus hand-off as the node (`ModalTerminal`'s `covered` into `useMdModeFocus`).
3. The chord reaches the modal through `window.nodeTerminal.onMarkdownToggle` only while it is the top dialog, while the canvas terminal and editor nodes' own subscriptions refuse the chord whenever a board is up (`lib/markdownChord.ts` `canvasOwnsMarkdownChord`; a hover flag can go stale under the opaque board, so one press could otherwise flip both).

### Comments and card metadata

The 💬 icon means COMMENTS on both surfaces (repurposed from the markdown view; ⌘M still toggles markdown or chat). On a terminal node it opens a right-side comments flyout (`.term-node__comments`, a sibling of the `overflow: hidden` root, hosting `BoardLogPanel` with `card: Pick<KanbanSession,'id'>`). In the modal it collapses and reopens the panel, which is open by default.

Under the modal header sits the card metadata strip (`CardMetaBar.tsx`): Members (assign), shown as colored initial avatars with a picker pool of me, live presence peers and board-log authors (name-keyed, no separate membership system), plus a Due date (`datetime-local`, red Overdue chip past due; cards show mini avatars and a due chip). Data is `kanban.meta [{nodeId, assignees, dueAt, priority}]` (priority low/medium/high/urgent as colored chips), read through tolerant readers (`cardMeta`), pruned with dead nodes, empty entries dropped.

### Board log

The modal's right third is the board log panel (`BoardLogPanel.tsx`, `state/boardLog.ts`): per-person comments and card activity from `<cwd>/.nodeterm/board-log.jsonl`, append-only JSONL (`core/board-log.ts`: tolerant newest-first parse capped at 500; text clamped to `BOARD_LOG_TEXT_MAX` 16 KB because an SSH append is ONE printf argument and ARG_MAX would silently drop it). Author is the presence identity. It is registered via `core/board-log-handlers.ts` in BOTH shells; the client sends only a projectId and the path always derives from the server's own registry, so no jail is needed.

- Events come from ONE pure funnel (`lib/boardLogDiff.ts`; binding invariant: its `cardTitle` argument returns '' for and ONLY for dead nodes; column deletion suppresses per-card moved-to-Ungrouped noise; prunes and reorders log nothing) plus `createNodeInColumn`'s card-created.
- Assign and due changes go through the same diff funnel (`member-assigned/unassigned`, `due-set/cleared`, `priority-set/cleared`). Agent-to-agent message deliveries are logged as `agent-message` by `agent-message-trace.recordDelivery` (`from`/`to` are node ids, `title` is the outcome). Unknown future event types render neutrally; the `BoardLogEvent.type` union in `shared/types.ts` is the source of truth.
- Feed rows show ABSOLUTE Trello-style stamps (relative in the tooltip).
- Local projects push changes via fs.watch; desktop SSH projects poll every 5 s while subscribed; inline projects show a hint.
- **Folding is a VIEW** (`lib/boardLogCollapse.ts`, `BoardLogFeed`): consecutive events by the same author (name AND colour) of the same type within two minutes of the run's NEWEST row render as one "×N" row that expands in place; the jsonl is never rewritten. The window is anchored, not chained, so a ×N never spans more than two minutes. Comments never fold, and neither do `agent-message` / `agent-read-cookies` (audit rows) or an issue's `run-started` / `run-ended` history (each row names a different session): `NEVER_COLLAPSE`.
- Deliberate v1 gaps: column-level events are stored but no card feed shows them; canvas-born nodes get no card-created; there is no card-deleted type.

Relay tabs BRIDGE boardLog to the host, with a pre-dispatch `sharedProjectId` scope guard in the relay dispatch: an out-of-scope projectId is refused before any registry or path resolution, and a connection drop replays its outstanding `onChanged` unsubscribes.

**The relay-guest scope jail is keyed on channel CLASS, not a per-feature list** (`main/remote/relay-project-scope.ts`). Every method whose name starts with `githubIssues:`, `board-log:` or `projects.` is project-scoped, and one the table cannot read a projectId out of is REFUSED on a scoped session. The switch it replaced had a `default: not project-scoped` arm, so a new verb in one of those namespaces reached another project's data with no refusal anywhere. A new channel in a scoped class therefore needs a table row to become reachable, and `relay-project-scope.test.ts` fails if a live IPC channel in a scoped class has none, so the fail-closed default cannot silently swallow a shipped verb.

**A comment can steer an agent**: the composer's @ picker (canvas flyout and card modal alike; both build candidates through `lib/boardMentions`) mentions an agent session on this board, and on send that session receives the comment through agent messaging; the row shows each delivery's outcome. A comment that ARRIVES in the log is display-only, forever. See [agent-messaging](agent-messaging.md).

### Handoff pings

`lib/handoffPings.ts`: the `assign` verb notifies a card's ASSIGNEE (this machine's presence name) when an agent files it into a `done` column or a `started` column past the board's first. Never on routine moves, and not for needs-you (the existing agent-status alert already covers every agent node). Same consent, background-only rule and per-node cooldown as the turn-end alert. A handoff ping arms a one-shot FOLD so the Stop hook that follows seconds later is not a second notification for the same moment. The fold lasts only while the user is away: a window focus in between drops it (`installHandoffFocusReset`), or the next chime, for a turn the user sat and watched, was swallowed. v1 is agent-driven moves only: a teammate's move arriving by git, or the phone's relay move, pings nobody.

## Keyboard

Board keys are registry commands in the `board` scope (group Board). It is the only scope allowed a bare letter or Space, because it resolves only while a board is up and carries neither `allowWhileTyping` nor `allowInTerminal`; that pair of refusals is the ONLY reason `normalizeBindingForCommand` lets it bind a bare key (the card modal's terminal, the comment box and the chat composer keep every key). Bindings:
- Space opens the focused or hovered card;
- J/K and ArrowDown/Up walk board order (the session cards on screen, column by column, Ungrouped first; GitHub cards excluded);
- ArrowLeft/Right jump to the same row of the neighbouring non-empty column;
- in the card modal, J/K step the modal.

Dispatch stays in Canvas's one listener. The mounted per-project board answers through `lib/boardKeys.ts` and DECLINES (the key falls through) when the focused control uses the key (`keyOwnedByControl`: Space on a button, link or checkbox, anything in a `<select>` or ARIA composite), when any dialog other than its own card modal is open, or when a card menu is up. Two bugs this had to fix: the canvas's CAPTURE-phase space-to-pan `preventDefault`ed every non-typing Space even while a board covered the canvas, so no board button could be pressed with Space (`spacePanKeydown` now takes `canvasCovered`); and a `Space` binding could never match because `e.key` is `' '` (`normalizeKey` maps it to `SPACE`). The Settings recorder captures a bare key only for a command that may have one (`board` scope or `allowBareKey`). See [keybindings](keybindings.md).

## Team progress

`lib/teamProgress.ts`, `components/TeamProgressChip.tsx`: a session that opened stations shows "N of M done" as a small ring on its board card, its card modal header AND its canvas node header. It is ONE component so three views of one node cannot count differently; clicking it lists the stations and a row travels to that node. A station is a session node that is the target of an OPENER rope.

Ropes carry two relations: "opened by" (`ctrl-<source>-<node>`) and a wait (`--after`, the verify panel), minted `ctrl-after-<dep>-<node>` (`waitRopeId`, `lib/edgeModel.ts`) and skipped. **The mark is what makes this hold, not rope order.** The canvas prunes every rope with an endpoint off the canvas, so deleting an orchestrator (or the user deleting its rope, or a `--project` open whose opener lives in another project) removed the opener's rope, and under the old "first rope is the opener" rule the first surviving wait read as the opener: a pipeline's upstream station showed the next one as its team, a verify panel's reviewed node showed its reviewers. Canvases saved before the mark are re-marked at LOAD by append order (`markLegacyWaitRopes`: every rope into a node after its first is a wait), which must run before the prune; one already pruned and saved has lost that evidence. A node that records its opener (`data.openedBy`, the station-notice stamp) closes that residual: only the recorded opener's rope may claim it, so a surviving wait is never promoted; the rope still has to exist (delete it and the station leaves the team, the same pair the station-failure notice reads). Nodes opened before the field keep the rope rule. Tests run the canvas's load, heal and prune steps before `stationsByOpener`.

Rules the count keeps:
- **Unknown is unknown, never done.** State is transient; after a restart a station reads `unknown` until it reports.
- Paused and hibernated count as a finished turn (both come only from an exit that refuses a working or blocked session).
- A CLI that announced its exit (`sessionEnded`) reads `ended` and counts (it is not running and never will be on its own; `unknown` would hold the ring below complete forever).
- A held launch reads `queued`.
- `done` plus `lastTurnError` reads `errored` (the `--after` verdict).
- A station that can never report (plain terminal, hook-less agent) is listed as "no status" and kept OUT of M (the line `--after` draws).
- A deleted station is not a station.

Subscriptions: each chip reads `teamProgressSig` (one character per station, no ids), never `byId`. The per-project board gets the teams as a prop from Canvas, the Omni lanes derive theirs from the stored `p.ropes` and `p.nodes`, and the canvas node header reads `useTeamStations` (`state/teamStations.ts`, a transient store Canvas publishes from its live control ropes). The previous map is threaded back into `stationsByOpener`, so an unchanged team keeps its array identity across drag frames and nothing re-renders. Desktop and Server Edition identical; Mobile: follow-up (the phone board would need the ropes).

## Omni Kanban (global swimlanes)

`components/kanban/GlobalKanbanView.tsx`: one swimlane per open project. It is off by default (`settings.omniKanbanEnabled`, in `settings.json`). `settings.omniKanbanAsDefault`, when true, makes `view.kanbanToggle` (⌘⇧B) open Omni; otherwise ⌘⇧B opens the per-project board. The `view.globalKanbanToggle` registry command (unbound, remappable) always opens Omni when enabled. State is `state/viewMode.ts` `globalKanban` (localStorage `nodeterm.globalKanban`, machine-local like `viewByProject`).

**Omni is a SCOPE of the kanban side, not a third view.** The view toggle (tab icon, ⌘⇧B, the menu) flips canvas and board; a board header's `KanbanScopeSwitch` ("This project" / "All projects", rendered only while the feature is on) moves between the two scopes. Leaving Omni through the switch lands on the project's board, and the view toggle from Omni lands on the canvas (it used to fall through to whatever the project's view happened to be, so "Canvas view" from Omni could land on a board). The decisions live once in `state/viewMode.ts` (`toggleBoardView`, `toggleAllProjectsBoard`, `showProjectBoard`, `showCanvas`), called by TabBar, the menu IPC and both registry commands. Omni has no close button. `globalKanban` deliberately stays independent of the active project, so a project switch made from a lane does not drop the user out of Omni. `isGlobalKanbanOpen()` is the single gate (fail-closed, static import of `useSettings`; the earlier `require` failed open in the packaged renderer).

- Non-active lanes are derived from serialized `p.nodes` via `toKanbanSessionState` (the persisted-state counterpart to `toKanbanSession`).
- The ACTIVE project's lane is fed LIVE by Canvas (`GlobalKanbanLive`: the same `kanbanSessionsFrom(nodes)` cards and live `teamStations` the per-project board uses), because its edits reach the store only at the next ~800 ms autosave. A store-fed lane reverted the card modal's controlled sticky textarea on every keystroke and pruned the assignment of a card created a moment earlier.
- The open card modal is ONE overview-level fact reported to Canvas's `setKanbanModalNode` (watched for Eco, wake-on-open, dictation target), never per-lane state.
- Each lane's board write resolves ITS project's session (`sessionForProject`) for the hosted read-only refusal and the board-log api.
- `pendingLaunch` never becomes `initialCommand` in the modal: the DAG launch must fire only when dependencies report done, and the canvas `TerminalNode` already delivers `initialCommand` via `writeWhenShellReady` after the `nodeterm:create-node` project switch.
- Active-project edits (rename, sticky, browser nav) route through Canvas live nodes (`setNodes` and `markDirty`), non-active ones through the store and `writeDisk`. Delete uses `ConfirmDialog` (not `confirm`) and then `deleteNodes` (active project) or `closeStoredNodes` (any other), the existing cross-project teardown funnel, never a hand-rolled copy.
- The top bar's project pills and Cmd/Ctrl+1..9 (`nodeterm:swimlane-jump`) jump to the lane; the header hint shows the correct modifier (`Cmd` on Mac, `Ctrl` elsewhere).
- Omni shows no GitHub issue lanes. Server Edition works as-is; Mobile N/A.

Server Edition for the whole board: pure renderer plus `workspace.save`, works as-is.

## Mobile (nodeterm-ios) board verbs

The phone reaches the board through three relay verbs, all landing in `WorkspaceStore.ensureRemoteBoard`, `setRemoteCardColumn` and `editRemoteCardLabels` (host-service `handleKanban`, wired in `main/index.ts`'s `hostBridge.kanban`, pure transforms in `core/project-kanban-write.ts`):
- `projects.ensureBoard` seeds the default columns on a project that has none;
- `projects.setCardColumn` moves one card;
- `projects.editCardLabels` adds, removes or creates board labels on one card (the phone's long-press label sheet).

There is ONE label model: the per-project palette in `kanban.labels` plus per-card ids in `kanban.meta[].labels`. The label verb writes it through the SAME transforms the canvas node's "+ Label" row and the kanban card use. The card-meta and label half of `lib/kanban.ts` moved to `@shared/kanban-labels` (re-exported, renderer call sites unchanged) so core can apply it, and a test pins that a phone edit produces the board `toggleCardLabel` produces. Params are validated at the write site (`parseCardLabelEdit`: bounded control-free ids, 1 to 60 character control-free names, colour from the closed palette, no id both added and removed) because they land in a git-shared, hand-editable file. A created name matching an existing label case-insensitively REUSES it (the picker offers no Create on an exact match). The first label on a board-less project seeds the default board, as the desktop's first "+ Label" does. A stale `add` answers `edited:false` with the CURRENT palette so the phone can redraw. Deleting or renaming palette entries is deliberately desktop-only (it touches every card). The iOS direct-SSH path has a Swift twin of the transform for projects on the host it dials.

Why the verbs are necessary rather than convenient:
1. The desktop board is a LAZY default (`kanban` is not written until the user's first board edit), so most project files carry NO board and the phone, which knows a project only by its file, could not offer one (measured: 1 of 13 project files on the author's machine had a `kanban` block). The default columns therefore live in `@shared/kanban-default-board`, read by `defaultKanban()` AND by the core seeder, and copied verbatim (under a pinning test on both sides) by iOS `KanbanDefaults`.
2. An SSH project's file is on a THIRD machine the phone has no credentials for. The verb writes the entry's `cache` and lets the ordinary mirror push it, exactly what a desktop card drag does, so it needs nothing new from `reconcileSsh` (that decides by `rev` and unions only `nodes`). Its cache change IS persisted to `workspace.json`, because for an ssh entry the cache is the local record.
3. The phone's older direct-SSH write inlines the whole `project.json` into one argv string and dies at Linux's `MAX_ARG_STRLEN` (this repo's own `.nodeterm/project.json` measured 114,695 bytes, about 15 KB under the 128 KB ceiling).

**Both verbs announce their write on `workspaceExternalChange` and that is not optional**: the renderer holds its own board and the next whole-workspace save serializes THAT, so a change the renderer never heard about is one the next autosave reverts.

iOS follow-ups: the phone must at least not break on `category`, `rank` or `views` (extra JSON keys its board decoder ignores). The relay-served move now lands at the top with a rank, while the phone's direct-SSH writer (`KanbanBoardWriter`) still appends without one; the next desktop write into that column ranks it, and array order already shows it correctly. `KanbanDefaults` should gain the three categories.
