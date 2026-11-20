# GitHub pull requests on the board

Pull request cards, CI and mergeability status, PR links to branches, issues and session cards, and the opt-in merge-driven card move. Core code: `core/github/graphql-pulls.ts`, `core/github/pull-status-tracker.ts`, `core/github/pull-memory.ts`; shared: `shared/github-pull-status.ts`, `shared/kanban-pull-links.ts`; renderer: `lib/pullLinks.ts`, `lib/pullAutoMove.ts`, `lib/pullChase.ts`. The shared sync budget and failure rules are in [github-sync](github-sync.md); the board is in [kanban-board](kanban-board.md).

## Pull request cards are harvested, not fetched (read-only)

`/repos/{repo}/issues` returns pull requests too (`client.listIssues` used to `continue` past them), so keeping them costs ZERO extra requests and inherits the incremental `since` watermark, the heartbeat ETag, the 60 s poll and the cache snapshot the issue lane already has. The alternative was measured and rejected: `/repos/{repo}/pulls` IGNORES `since` (a day-old `since` returned the same 100 items as none), each item is about 25 KB against about 7 KB, and it would be a second ETag, paging and cache lineage, for `head`/`base` and nothing else (mergeable, reviews and checks are per-PR legs either way).

The harvest's fields are `draft` and `pull_request.merged_at` (the only thing separating merged from closed, since both report `state: 'closed'`), plus the labels and assignees the issue shape already carries. There is no `head` in that payload: `GitHubPullMeta.head` stays undefined until something asks per branch (`/repos/{repo}/pulls?head=owner:branch`), which is one request per question rather than a field on every poll.

Three rules the harvest brought with it:
1. One snapshot now holds both kinds, so `GitHubIssueQuery.kind` (absent = `'issue'`) keeps the issue lane's items and counts byte-identical to before. `moveIssue` refuses a PR by number (`invalid-target`), because the two share a number space and its membership check alone would hand one to a write path that cannot serve it.
2. `MAX_ISSUES` and the 64 MB bound are shared, so an overflow **evicts pull requests first, oldest-updated first** (`evictPullsToFit`) and marks the snapshot `pullsTruncated`. The existing `incomplete` read-only path fires only if the issues ALONE still miss the bound. A repository large enough to overflow degrades in the new half, never in the board it already had. An incremental pass carries the flag forward (it never re-fetches what it dropped), so only a full reconciliation may clear it.
3. The `pulls` source is `readOnly` in the source registry: no drag, no move control, and its page reports `readOnly: true` on the wire rather than trusting every consumer to remember.

## CI, mergeability and links

The issues harvest cannot say a PR's head, checks or mergeability, so ONE GraphQL read per repository adds them: every open PR's `headRefName`, `headRefOid`, `isCrossRepository`, `isDraft`, `mergeable`, `mergeStateStatus`, the head commit's `statusCheckRollup`, `closingIssuesReferences`, plus the 30 most recently merged or closed PRs (so a branch link survives the merge). MEASURED on this repository (60 open PRs): **1 point** of the separate `graphql` budget, about 18 KB; the per-PR checks read (modal open only) is also 1.

### When it runs

After a heartbeat that reported a change, on a user refresh, on the first heartbeat of an app run, and when the last read failed or the budget skipped one (`owed`). Never on a 304 otherwise.

A finished check run does NOT move the heartbeat, so an UNDECIDED PR (mergeable UNKNOWN, or rollup PENDING/EXPECTED) is CHASED. Measured: 40 of 50 open PRs read UNKNOWN on a first read and all settled 20 s later, because the first read is what starts GitHub's computation.
- Schedule: 30 s, 1 min, 2 min, 5 min; at most 12 reads per episode; only while a board is VISIBLE.
- The renderer asks (`githubIssues.chasePulls`) every 15 s while `document.visibilityState` is visible; the host answers from a map. Schedule, cap and the synchronous `claimChase` live in core, and no context (credential chain) is resolved unless a read is due.
- A new undecided PR or a new head starts a new episode; the same stuck PR does not restart the count.

### Semantics (each a shipped bug somewhere)

- A null rollup is "no checks" and renders NOTHING (never a tick).
- Only `mergeStateStatus === 'CLEAN'` is "Ready to merge" (MERGEABLE+BLOCKED is real: 2 of 31 here).
- A rollup counts only at the current `headRefOid`, and a CI result is never carried from an older head.
- A failed read keeps the last snapshot marked stale (`pullStatusFreshness`, greyed after 15 min).
- A FORBIDDEN or INSUFFICIENT_SCOPES answer for the rollup or mergeability HIDES that region (`access`), which is not the same as a null rollup.
- Enum values (`mergeable`, rollup `state`, `mergeStateStatus`) decode LENIENTLY: a value GitHub adds later claims nothing and is not chased, and an unrecognised rollup state is `UNRECOGNIZED`, never `null` ("no checks"). One new value must not fail every read of the repository (it happened with `state_reason: duplicate`).

### Budget

GitHub meters `graphql` apart from `core`, and so does the coordinator: `throttle(identity, at, resource)`. A primary limit is tagged with its resource (`GitHubClientError.resource`, from `x-ratelimit-resource`, or GraphQL's 200-with-`RATE_LIMITED`) and holds only that resource; an untagged (secondary) limit still holds the identity. A spent graphql budget (the user's own `gh pr list` spends it) must not stall REST issue sync.

### Links

- **PR to issue** is GitHub's own `closingIssuesReferences` (same repository only), deliberately NOT unlinkable on the board: GitHub closes the issue at merge whatever the board shows.
- **PR to session card**: the PR's head equals `data.worktree.branch` of the card's nearest bound group (`worktreeBranchOf`, no git read; a stale binding still names the branch, which is exactly when its PR merges). A FORK PR never links (36 of 50 open PRs here are forks).
- **Unlinking** writes a git-shared tombstone in `ProjectKanban.pullLinks`. It is a BOARD-LEVEL field on purpose: every card-meta setter rebuilds `meta[]` entries from a fixed field list, so a field added there is erased by the next member, due or label edit.
- On an SSH project NO card carries a worktree branch (`kanbanSessionsFrom` leaves it off where the cards are built), so the card face, the move and the modal (which states the worktree reason) cannot disagree.
- **PR to issue to session card**: a session started on an issue (`data.issueRef`, see [board-dispatch](board-dispatch.md)) links to every PR whose `closingIssuesReferences` name that issue, matched against the pull board's own `repository` (a `closes` number means nothing in another repository). Here a FORK PR does count, since GitHub's "Closes #N" is meaningful wherever it comes from. The host memory remembers what a PR closed while open, so the link survives the merge that should move the card; the same tombstones apply, and an issue-bound card on an SSH project still links this way (only the branch half needs a worktree group).

## What this machine observed lives on the host

`core/github/pull-memory.ts`, persisted beside the issue cache per identity and repository, deleted with it. It holds every PR this machine has seen, its head, its lifecycle, whether it was seen open, and `mergedSeenAt`: the first time it was seen merged AFTER being seen open (an OBSERVED merge).

The read lists only 50 open and 30 recent PRs, so without memory a closed-unmerged PR's block would expire once 30 newer PRs closed. Remembered PRs stay on the pull board (closed ones until unlinked, merges for 30 days) and an unlisted one takes its lifecycle from the REST harvest.

The first version kept a per-card "seen" map in `settings.json` instead. Review found three failures in that shape: the planner's output exceeded the sanitizer's bounds so the hook rewrote settings in a loop until React threw; the block expired; and two Server Edition tabs both moved the card while a background tab's settings write reverted another tab's changes.

## Merge-driven move (session cards only, OFF by default)

The switch, the target column and `armedAt` are MACHINE-LOCAL (`settings.kanbanPullAutoMove.projects[projectId]`, written ONLY by the user's own Settings action). It makes this machine write the shared board on its own, so a switch in the project file would make every clone move and commit cards nobody on that machine asked for. The per-card opt-out is board content (`pullLinks.noAutoMove`).

Guards in order (`decidePullAutoMove`):
1. opted out: never;
2. any linked PR open or draft: wait;
3. any closed unmerged: blocked until the user unlinks it;
4. already in target: nothing;
5. no linked merge with `mergedSeenAt >= armedAt`: no move (arming never sweeps old merges; `armedAt` is taken from the HOST's clock via the pull board's `now`, because `mergedSeenAt` is stamped there and a Server Edition browser's clock can be off).

Each planned move must then win the host's one-time CLAIM (`githubIssues:claim-pull-auto-move`, persisted in the same memory). The first ask across every window wins, and a card dragged back is not moved again for the same merges.
- **Claims are recorded PER PR** (project + card + PR), never per PR set. The linked set changes on its own (a merged PR ages off the pull board, another PR on the branch joins), and a set-keyed claim read every such change as a new transition and moved a dragged-back card again. A claim is granted only for a PR that has not moved this card yet; set keys an earlier build wrote are read as a claim on each PR they list.
- The board asks each (card, PR set) ONCE while it is in flight or after it was won, and re-asks a REFUSED one only after `REFUSED_CLAIM_RETRY_MS` (60 s: the host also refuses transiently, before it is bound to the project or while it clears its cache, and remembering that for the board's lifetime lost the move). A failed call is forgotten. It re-plans only when a field the planner reads changes; it used to send a claim per canvas change for a dragged-back card.
- **The claim is refused unless THIS card was noted waiting on one of those PRs while it was open** (`githubIssues:note-pull-waits`; the host records a note only for a PR it holds as open itself). `mergedSeenAt` is a fact about the PR, and without the per-card note a card that first appeared after the merge (a follow-up terminal in the same group, a teammate's card by git pull, an issue-bound session started later) would win a fresh claim and jump to Done.
- The move is applied as a compare-and-set on the column the decision saw (`applyPullAutoMove`, run by Canvas against the store's latest board), writing ONE `card-moved` board-log line whose `title` names the PRs. The planner writes nothing; it runs only while the board is open and never on a stale snapshot.
- **GitHub issue cards are never auto-moved.** GitHub already closes them when a `Closes #N` PR merges, and a second writer would race it and could clobber `state_reason`.

## Surfaces

Desktop and Server Edition identical (core plus renderer). `githubIssues:pull-status`, `:chase-pulls`, `:pull-checks` and `:claim-pull-auto-move` are registered by the shared core handlers and served to relay tabs through the project-scope table (see [kanban-board](kanban-board.md)); a relay tab never auto-moves, because the board belongs to the other machine. Check detail is bounded for relay guests: one read per PR per 15 s, ten per project per minute, both decided before a credential is resolved. **Mobile: follow-up.** The phone board carries session cards only; showing PR CI there means carrying `GitHubPullBoard` over the relay dialect.
