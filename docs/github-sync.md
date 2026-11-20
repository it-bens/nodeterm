# GitHub sync

How the desktop and Server Edition core talk to GitHub for the kanban board's issue lane: what a poll costs, how failures are classified, how the rate budget is read, and what a write needs. All of it is core (`src/core/github/*`) and identical on Desktop and Server Edition. The board UI is in [kanban-board](kanban-board.md); pull request cards, CI and merge-driven moves are in [github-pull-status](github-pull-status.md); agent runs bound to issues are in [board-dispatch](board-dispatch.md).

## Before adding a GitHub call

Check what the existing poll already fetches. Pull request cards needed no new request at all: `/repos/{repo}/issues` returns pull requests, and the client used to discard them. `/repos/{repo}/pulls` looks like the obvious endpoint and is the expensive one: it ignores `since`, so it can reuse none of the incremental machinery, and its items are about 3.5x the bytes (measurements and the eviction rule that keeps the issue lane unaffected are in [github-pull-status](github-pull-status.md)).

Rules for any new GitHub call:
- **Never decide what a failure means yourself.** Pass it through `classifyGitHubFailure` (`core/github/failure.ts`). Only `unauthorized` may ever read as "signed out"; a rate limit, an outage or a dropped connection must say so instead, or the user re-authenticates an account that is fine.
- **Go through the request coordinator and a client built by the host.** That is what feeds every response's rate budget to the coordinator, pauses background work below the floor and caps waits. A request that bypasses it is invisible to the budget. Prefer a conditional request (`If-None-Match`) for anything you poll: a 304 is free.
- **A write whose meaning comes from the project file needs `context.mappingApproved`.** The column mapping is git-shared; approval covers it, and reads do not need it.
- **Only a completed scan moves the incremental cursor.** `lastSuccessfulRefreshAt` is the next scan's `since`; a write that folds one issue into the snapshot must leave it alone, or other people's changes from before the write wait for the daily full pass.
- **GraphQL spends a different budget.** GitHub meters `graphql` apart from `core`; ask the coordinator with the resource (`throttle(identity, now, 'graphql')`) and let a primary limit carry its `resource`, or a spent GraphQL budget stalls REST issue sync. A GraphQL field the token may not read comes back `null` plus a FORBIDDEN error: decode that as "hidden", never as its empty value (a null check rollup alone means "no checks").

## Seven sync rules

### 1. A poll that finds nothing spends nothing

Before each pass, `client.issuesHeartbeat` asks for the single most recently updated item (`state=all&sort=updated&direction=desc&per_page=1`) with `If-None-Match`; a 304 skips the scan. MEASURED against this repo (read-only `gh api`): twenty 304s moved `x-ratelimit-used` by 0, the next 200 by 1; five plain 200s by 5. The `since` scan could never do this itself, because `since` moves every pass, so its URL (and any ETag) never repeats; the snapshot's `etags` map was declared and always written empty.

- The validator is the one read BEFORE the scan (a change landing mid-scan stays "new"), it is persisted in `snapshot.etags.heartbeat` (a restart does not pay a full read), and a 304 NEVER skips a full reconciliation (a deletion or transfer does not move the top item) or an incomplete repository. It covers pull requests by construction: same endpoint, same `updated_at` the scan filters on.
- **Only a completed scan advances the incremental cursor** (`lastSuccessfulRefreshAt`, the next scan's `since`). A board write folds its one confirmed issue into the snapshot and leaves the cursor alone. It used to set it to the write's time, so a third party's change landing between the last scan and our write fell outside the next `since` window until the daily full pass.
- **A 304 still prompts the board to re-read** (an empty delta, served from the local cache, no GitHub cost), exactly as every successful refresh does. A page is not only issues: read-only state, the mapping approval and the completion column are derived by the host at query time. The first version, which emitted nothing on a 304, left a board read only after its user approved the mapping. Approve and revoke also notify the project's open boards at once (`service.notifyProject`).
- **Unknown `state_reason` values decode as no reason.** GitHub added `duplicate` (one of cli/cli's last 100 closed issues, measured 2026-09-29), and the strict decoder failed that repository's whole scan as malformed, so it never synced.
- **The credential check is conditional too** (`createTokenValidator`). Every poll re-resolves the credential (30 s memo, 60 s poll), and each resolve was an unconditional `GET /user`, one real request per poll even after the heartbeat. With `If-None-Match` an unchanged identity is a free 304 (MEASURED: ten conditional reads, +0), and a bogus token presenting a valid validator still gets 401. Idle polling costs zero quota.

### 2. The budget is read from every response and acted on before GitHub refuses

`client.onRateLimit` reports `x-ratelimit-*` from 200, 304 and errors alike to `GitHubRequestCoordinator.noteRateSample`, keyed by the credential's identity (the host builds the client from token and userId); within one window the LOWEST reading wins.
- A spent `core` budget blocks new requests at once.
- Below `backgroundFloor(limit)` = max(100, 10%) the BACKGROUND poll stops until the window resets (the budget is the whole account's: `gh`, the browser, other tools), while a refresh the user asks for still runs within its own floors.
- **Every wait is capped** (`MAX_RATE_WAIT_MS`, 10 s in TOTAL): past it the request is refused at once with its retry time, instead of holding one of four read slots and an IPC call for up to an hour.
- The page and the Settings status carry the `throttle`; both say "held until HH:MM" (`lib/githubSyncStatus.ts`), and subscribers are prompted once per deadline, not per skipped minute.
- Do NOT add a `/rate_limit` probe. Measured for the same token in the same second it answered 5000 left / used 0 with a different reset time than the response headers (4968 / 32). The headers are the truth.

### 3. Throttled is not signed out

`classifyGitHubFailure` (`core/github/failure.ts`) is the ONE classifier: only a 401 or a non-rate-limit 403 is `unauthorized`; network, timeout, 5xx and malformed bodies are `unreachable`; limits are `rate-limited`.
- Token validation is tri-state (`TokenValidation`). The resolver keeps the last credential GitHub vouched for (SAME token only) through an unknown answer, and with none throws a `GitHubReachabilityError` rather than returning the null that every caller reads as "sign in".
- Auto does not fall through to the saved token when the CLI's token merely could not be checked, because that would switch identities for the length of an outage.
- **`gh auth status` is never run.** Measured on gh 2.45 with GitHub unreachable it printed "The token in hosts.yml is invalid". `gh auth token` only reads the local store and our own classified `/user` check decides.
- Settings says "GitHub could not be reached to check the sign-in" and keeps the last confirmed sign-in on screen; `saveToken` refuses an unchecked token without calling it invalid.

### 4. A close carries its reason

`UpdateIssueInput.stateReason`: the close confirm offers Completed / Not planned (Completed preselected, GitHub's default); a reopen sends `reopened`. It rides ONLY with a state change (the "send `state` only when it changes" rule stands), the client refuses a reason of the wrong kind, and a close GitHub recorded with a different reason is not reported as confirmed.

### 5. Writes need an approval that covers the column mapping

The mapping (which label a column applies, which column closes) is in the git-shared `project.json`, so a pulled commit could re-aim writes under an approval given for something else. An approval records `githubMappingDigest(repository, completionColumnId, columnMappings)`. Column ORDER and titles are excluded: they change nothing GitHub sees, and `config.revision` includes order and would make every column drag revoke writes.
- Moves and "Create missing labels" require the digest to match (`context.mappingApproved`). READS stay on the repository approval, since the mapping only decides what a write does.
- A mismatch makes the board read only and says why (`page.mappingNotApproved`); Settings offers "Approve column labels".
- Upgrade: an approval from before this change has no digest. It keeps reading and must be approved once more before the board writes.
- Pinning the on-disk mapping at first load was rejected: it would silently trust whatever a `git pull` delivered between the upgrade and that load.
- Because titles are outside the digest, dispatch consent binds them separately (see [board-dispatch](board-dispatch.md)).

### 6. Revoke deletes the cache

The plaintext issue cache (bodies included) is removed by the same bounded path as "Clear cached data", AFTER the revoke is recorded. A failed delete is reported (`revoked-cache-kept`), never left approved. The cache file is per (identity, repository), so another project on this machine bound to the same repository re-fetches too.

### 7. Every client that leaves releases its subscription

The service polls every 60 s per subscribed repository. The desktop window releases on `closed`, `render-process-gone` and `did-navigate` (`main/renderer-client-release.ts`; not `did-start-navigation`, which also fires for a navigation the guard blocks), beside the relay peers and Server Edition sockets that already did.

## A GitHub issue reaches a pane only as a validated reference

Issue titles and bodies are written by strangers on public repositories, and a launch line is typed into a shell. Anything that starts or instructs an agent about an issue goes through `@shared/github-issue-ref`: `issueLaunchPrompt` (the only composer, which re-validates `owner/repo#N` itself) for text, and `normalizeIssueRef` wherever a stored `issueRef` is read (it comes from a git-shared file). Never interpolate `issue.title` or `issue.body` into a prompt, and never add a path that posts an agent's output to GitHub on its own: posting is public, and only the user asks for it. Details of the bound-session flow are in [board-dispatch](board-dispatch.md).

## Surfaces

Desktop and Server Edition are identical (all core). **Mobile: N/A.** The phone's board carries session cards only (the `githubIssues:*` channels are served to relay TABS, never the phone dialect), so nothing there can read a throttle as signed out; surfacing GitHub cards on the phone would need this whole contract carried over the relay.
