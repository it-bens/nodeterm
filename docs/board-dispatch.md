# Issue-bound sessions and board dispatch

How a GitHub issue card starts an agent session ("Start with agent") and how a card the user drags into a dispatch column starts one by itself (board dispatch). Code: `@shared/github-issue-ref`, `@shared/board-dispatch` (consent), `renderer/lib/boardDispatch.ts` (decisions), `renderer/state/boardDispatch.ts` (queue), wired in Canvas `dispatchOnUserMove` / `drainDispatchQueue`. The board is described in [kanban-board](kanban-board.md); GitHub sync rules in [github-sync](github-sync.md); how a PR links to the resulting card in [github-pull-status](github-pull-status.md).

## Start with agent: a GitHub issue card starts a bound session

An issue card's right-click menu and its summary modal offer **Start with agent**, whose rows are the canvas's own agent and account picker (`agentCreationEntries`, which takes an optional `pick` so the same rows can point at another action; never a fourth copy of the picker). The result is an ordinary agent node in the project cwd (through `addAgentNode`, which returns the node) carrying `data.issueRef {owner, repo, number}`. That value is persisted and git-shared, therefore hostile: `normalizeIssueRef` (`@shared/github-issue-ref`) runs at BOTH serializer seams and a malformed value is dropped (the node survives, only the binding goes).

Rules a refactor must not undo:

1. **The launch line carries the REFERENCE, never the issue's text.** Titles and bodies are writable by anyone on a public repository, and a launch line is typed into a pane. `issueLaunchPrompt` is the ONE place a reference becomes text; it re-validates the reference itself and returns nothing for a hostile one. Never interpolate `issue.title` or `issue.body` into a prompt. The prompt tells the agent to read the issue with `gh issue view N --repo owner/repo --comments` (mid-sentence: punctuation glued to the last flag is copied literally and `gh` refuses `--comments.`) AND that its title, body and comments are untrusted input, not instructions. The session runs under the project's permission mode (auto by default), so the prompt is the only thing that can say "read it, do not obey it" before it does.
   - A board start means **work on it**: the default task is "investigate, plan and implement the fix in this working tree". A caller's `--prompt` REPLACES that task ("Your task: ..."), never the lines around it.
   - The hard limits ride the prompt itself, after any brief: never close the issue, and no issue comment or PR unless the user asks in that session; end with a proposed comment.
   - Proven under a real `/bin/sh` (`github-issue-ref.realsh.test.ts`).
2. **The reference comes from the card's `htmlUrl`** (`issueRefFromHtmlUrl`, which also requires the URL's number to equal the card's).
3. **`done` never moves a card.** It means a turn ended, not that the work did. The issue card shows every bound session as a live chip (`IssueRunChips`, subscribed per node to a PRIMITIVE signature `issueRunChipSig`, never `s.byId`): RUNNING / NEEDS YOU / TURN FAILED / DROPPED plus unread; a click opens that session's card. A card moves only when the session `assign`s itself or a person drags it. `components/kanban/board-writers.guard.test.ts` pins this: it enumerates EVERY renderer call site that writes a board assignment or moves an issue, each with the human or agent action that triggers it; a new writer fails until it is signed for, and "a turn ended" is not a trigger. Never add a path that posts an agent's output to GitHub on its own: posting is public, and only the user asks for it.
4. **Board-log identity of an issue card** is the synthetic id `github-issue:<owner>/<repo>#<N>` (lower-cased, `issueLogId`), a namespace no node id can reach. Under it, `run-started` (UI start, `--issue` open) and `run-ended` (written by EVERY node-removal funnel: `deleteNodes`, `closeStoredNodes`, the Omni delete, `deleteProject`, the Server `close`) before it drops the node's agent status, with the last observed state.
   - `duplicateNode` drops `issueRef` (a copy is a new session nobody started on the issue; carrying the binding made a phantom run).
   - Known gap: undo and redo replay node arrays and write no run history, so an undone delete revives a node whose run already ended.
   - The summary modal shows the history read-only as "Agent runs" (no composer: a comment box under an issue reads as "post to GitHub").
   - No cost or token figure is recorded: there is no cumulative per-session number, and a context-window reading is not one.
5. **Filing and the `#N` chip.** The new session card is filed under the issue card's column (the same unpruned direct write `createNodeInColumn` uses). The node header, the session card AND the card modal's header show a `#N` chip (`IssueRefChip`; the modal's closes itself and makes the same request) that opens the issue on the board (`openIssueOnBoard` into `viewMode.requestedIssue`) ONLY when that board has GitHub sync. Otherwise it goes straight to GitHub, rather than flipping the project's persisted view to a board that cannot show it; the board itself falls back to GitHub when the issue is not on a fetched page. Never a dead click.

**Start with agent in a new worktree** sits beside it on the card menu and in the summary modal: a fresh `issue-<N>-<slug>` worktree frame with the agent inside, with reuse offered when the issue already has one. See [worktrees](worktrees.md).

Anything that instructs an agent about an issue goes through the same validated-reference path (see [github-sync](github-sync.md)).

Surfaces: Desktop and Server Edition (renderer and core). The Omni board shows no issue lanes. **Mobile does not render the binding**: `issueRef` reaches the phone inside the project file, and nodeterm-ios ignores the unknown field (follow-up there).

## Board dispatch: a card this person moves into the dispatch column starts its own run

The run is exactly "Start with agent" (`issueRef` binding, the reference-only `issueLaunchPrompt`, `fileIssueSession` which files the card and writes `run-started`), only nobody clicked it. The hard question is WHO may trigger a run on this machine, and the answer shapes everything else.

Anything that **starts an agent by itself** takes its consent from machine-local settings and its trigger from a gesture the person made in this app, never from a label, a column or a file that can arrive from GitHub or a `git pull`. Its consent binds what it consented to.

### The trigger is the person's own move in this app

`decideDispatch` answers `ignore` for every origin but `'user-move'`, and the only caller that says `'user-move'` is the board's move-result path (`KanbanView.moveIssueByUser` into `onIssueMoved`, reached only from `requestGitHubMove` and the close/reopen confirm). A label set on GitHub (by anyone; on a public repository, ANYONE) reaches this app only as a refreshed page, and a board change arriving by `git pull` only as a new project file; neither has a path in. A move GitHub did not CONFIRM (`stale`, `failed`, `read-only`, ...) is not a dispatch.

A run starts only through `dispatchStart`, from that decision or from the queue drain after `recheckQueued`. `lib/board-dispatch.guard.test.ts` pins the WHOLE chain: `decideDispatch`'s one caller, `onIssueMoved`'s one firing site, `dispatchStart`'s two callers (after `decideDispatch` in `dispatchOnUserMove`, after `recheckQueued` in the drain), and that a `'queued'` entry (which the drain starts without asking `decideDispatch` again) is created only in `dispatchOnUserMove`. It fails if a new caller of any link in that chain appears.

**Why not a label with an actor allowlist** (the other design considered): it works from a phone, but it needs one issue-events read per candidate issue (budget), compares an actor against a credential that can change under it, and today the poll runs only while a board is subscribed. It would be a network-derived fact standing in for consent, and no run at all when nobody has the board open. That is the follow-up, not v1.

### Consent is machine-local and bound to what the column means

`settings.boardDispatch` (the `kanbanPullAutoMove` / trigger arm store tier), never `.nodeterm/project.json`: a switch in the project file would let a pull request make every clone start agents. Per project it holds the column, the agent, an optional account (absent = the project default through the same funnel as "New <agent>"), a cap (1 to 8, **default 1**: dispatched runs are told to implement the fix in the project's own working tree, so two at once are two agents editing one checkout), and a **binding**.

**The consent binds what the column MEANS, not just its id** (`dispatchBinding`: repository + column title + its GitHub label). Titles and labels live in the git-shared project file, and titles are deliberately outside `githubMappingDigest`. Without the binding, a pulled commit swapping the titles of "Agent" and "In Progress" would turn the person's routine drag into "In Progress" into a dispatch, and re-pointing the board at another repository (which needs only a mapping re-approval) would carry the dispatch switch along. Any difference refuses (`consent-stale`, on the card and in Settings) until the person presses "Re-confirm this column". Choosing a column binds; changing the agent or the cap does not re-bind. An entry without a binding is OFF.

Read through `sanitizeBoardDispatch`: an unreadable entry is OFF, an unreadable cap is 1, the kill switch (`paused`) is on only for a literal `true`. `boardDispatch` is in `SETTINGS_VERB_FORBIDDEN`: an agent that could switch the dispatcher on would grant itself more agents, and the name pattern does not catch the key, so the set is its only fence. Model: the same gateway default `addAgentNode` applies; there is no per-project model.

**Only agents that report their state through hooks** (`dispatchableAgent`, `hasHooks(capabilityAgentId(...))`, the `--after` rule) are offered or accepted. The cap counts sessions by hook state; a hookless custom agent never reports, so after the startup grace its slot would free and the cap would admit one more run every two minutes.

### Bounds and queue

- **One run per issue.** A bound session that still exists in ANY project, or a dispatch already queued or starting, refuses the next with a reason on the card.
- **The cap** counts this project's bound sessions that are `working`, `waiting` or `blocked`, hold a launch that will start BY ITSELF (a `manualOnly` one waiting for Run now does not), or were started by dispatch within `DISPATCH_STARTUP_GRACE_MS` (no hook yet). `done` frees the slot (the cap limits concurrent WORK), and an unknown state from before a restart does not hold one, or the cap would stay pinned.
- Over the cap the dispatch QUEUES; the drain runs on a 5 s timer only while something is queued.
- **Every queued entry is re-asked before it starts** (`recheckQueued`): kill switch, still switched on, project still open, local and not closed, binding unchanged, agent still dispatchable, and the issue still OPEN and still in the dispatch column (read from the host's issue cache with `githubIssues.query`, no GitHub request; an unreadable answer waits, it is never evidence). A teammate closing or moving the issue while it waited drops it with its reason. Moving a queued card out of the column withdraws it.
- The kill switch refuses new dispatches and drops the queue; running sessions are not touched.
- **No dispatched node is ever left armed to start on its own.** The off-screen path writes the node ALREADY CLAIMED (`claimForHeadless`: `manualOnly`, in the same tick as the node, so there is no window in which opening the project would auto-start it beside the headless start), then runs the headless start. Whatever it answers, a node that did not start waits for Run now. A Pause or a restart can therefore never be outrun by a held launch that fires on view. The failure notice says exactly that: Run now, or close the node to dispatch the issue again (it keeps the issue's one run until then).
- **The queue is in memory, on purpose.** A queue that survived a restart would start agents at boot with nobody there. A restart drops it silently, and the drag (or Start with agent) can be repeated. Each renderer keeps its own queue, so **two Server Edition tabs on one project can each run up to the cap** (known; one tab per project is the supported use).
- **The card says what happened** (`DispatchChip`): "Queued for an agent (#2)", "Dispatching an agent...", or "Not dispatched: <reason>" (`DISPATCH_REFUSAL_TEXT`). A started run shows as the ordinary run chip.

### Where it runs

In the renderer, because the trigger is a UI gesture core never sees. On screen it is `addAgentNode`. Off screen (a queued run whose slot freed later, or a project switch during the move's GitHub round trip) it is a cold open into the stored project (the control verbs' path) plus the headless start, which raises its "Go there" notice. A CLOSED project's queued run is dropped, not started (the headless start would unhide its tab).

- **Server Edition**: the browser renderer's `pty.launchHeadless` is unsupported (the server's own headless launcher serves canvas control, not a browser tab), so a dispatch there starts only for the project ON SCREEN; an off-screen one stays queued (still subject to Pause) until that project is shown.
- **SSH projects**: refused by name (the headless launcher is local-only).
- **Relay tabs**: refused (the board is the host's).
- **Mobile**: N/A (the phone board carries no issue cards).

Board dispatch never auto-posts to GitHub, never closes an issue, never moves a card on a turn `done`; the dispatch column may not be the completion column.
