# Usage indicator

The usage pill and popover: what the indicator is scoped to, how Claude, Codex and Grok usage is read locally and over SSH, how failures render, and how the bottom chrome shares its width. Main code paths: `renderer/lib/usageScope.ts`, `core/usage/` (`claude-usage.ts`, `remote-claude-usage.ts`, `remote-codex-usage.ts`), `CanvasPills`. Per-account identities are described in [managed accounts](managed-accounts.md). SSH Codex device checks are in [Codex SSH metrics](codex-ssh-metrics.md).


- **Active Claude organization** (#552), local Desktop and Server usage snapshots carry optional
  `organization` metadata from the SAME account's `.claude.json` (system, managed or linked).
  Read it even if Keychain/file credentials already have an email; unreadable/missing metadata
  preserves the email and usage. A known email mismatch drops metadata. Managed usage cannot use
  an unscoped Keychain token: a matching email alone does not prove it belongs to the same org.
  The popover names the org beneath the email; internal type/tier/id remain tooltip details.
  Refresh re-reads metadata together with usage; normal cache/poll intervals still apply.
  SSH's existing shell reader does not supply organization metadata and keeps its email-only
  fallback. Mobile's usage mirror currently omits it; displaying orgs there needs a follow-up
  mirror/iOS protocol change. No organization picker, credential writes or switching are added.


- **Bottom chrome shares a measured width budget** (issue #853). `CanvasPills` observes the canvas
  wrapper and actual dock, bounds the left row with an 8px gap, and uses a row above the dock when
  fewer than 200 CSS pixels remain. Rects are converted back through UI scale. Usage summary text
  ellipsizes; refresh never shrinks. Do not clip the whole row or give it a stacking context:
  usage/RAM popovers must escape independently above the sidebar and board. Desktop and Server
  share this renderer. The real-browser regression is `scripts/usage-layout.test.ts` (`CHROME_BIN`).


- **The usage indicator is scoped to the ACTIVE project** (`renderer/lib/usageScope.ts`, pure +
  unit-tested), it describes **the machine that project runs on**, and nothing else. A local
  project shows this machine (system + managed local accounts + the billing providers, whose
  credentials are all local); an **SSH project shows that host's Claude and Codex accounts**, no local
  Claude, no local providers, no other host. Without this the panel showed every source at once:
  each addition was individually reasonable and the sum was unreadable, numbers from three
  machines sharing one line with nothing saying which was which. Deliberately NOT narrowed to the
  project's `defaultAccountId`: the local side lists every local identity, so the machine is the
  scope and the account is a row within it. The pill spells out the scoped machine's **system**
  account (falling back to the first identity with data, so a host used only through a managed
  login isn't blank), managed accounts stay popover-only, the rule the local side always had.
  `usageScopeKey`/`scopeFromKey` exist because the active project object is rebuilt on every node
  serialization: the zustand selector returns ONE primitive so the indicator doesn't re-render on
  every canvas edit. ⟳ refreshes only what is on screen, and `usage.remote({hostKey})` reads only
  that host (cache eviction still runs against the FULL target list, so switching between two SSH
  projects doesn't throw each host's cache away).


- **Grok billing failures** retain per-view HTTP status or a safe timeout/network/invalid-response category in `ProviderUsage.diagnostics`. The default view still runs after a credits failure; recovered limits keep their diagnostic. Only two successful empty views imply no quota. Never include raw exceptions, URLs or response bodies, or refresh/write credentials. Desktop and Server share the core reader and popover; provider-only errors must keep the pill visible.


- **Usage failure readouts**, an empty Claude snapshot with `status: error` says "Could not
  read usage." in both the single-account and multi-account popovers, including beside healthy
  provider rows. Nonempty snapshots retain their last-known bars on error; no error-specific
  authentication advice is inferred from the status.
  **Local Codex reads (issue #1168)**: an HTTP 429 from the backend returns `rateLimited` and
  starts a per-home cooldown (`Retry-After`, else 2 min doubling, 30 s–1 h); during it NEITHER
  tier is asked, the app-server fallback reads the same account's limits, so falling through
  after a 429 spent the budget twice. The service holds each account row's last good numbers
  (`holdLastGoodProvider`). The SSH Codex reader has no cooldown yet.


- **Remote usage** (SSH hosts, `src/core/usage/remote-claude-usage.ts`), the source behind the
  SSH scope above. v1 excluded remote accounts, which left a user whose Claude only ever runs on a
  server staring at an empty indicator while the host had perfectly good numbers.
  **The token never leaves the host.** The desktop could `cat` the remote `.credentials.json` and
  call the API itself, it already reads remote transcripts over the same master, but a bearer
  token pulled off a (possibly shared) server into another machine's memory buys nothing: the host
  can make the request itself. So core generates a POSIX **sh+curl** command, the shell runs it
  over the project's ControlMaster, and only the JSON answer comes back. Three details are
  load-bearing:
  1. **The token is piped into `curl --config -`, never `-H` on the command line**, argv is
     world-readable via `ps` on a shared host.
  2. **`.credentials.json` holds more than one `accessToken`**, every MCP server the CLI has
     authorized keeps its own under `mcpOAuth`. The extraction narrows to the `claudeAiOauth`
     object first (exactly as the local `parseCreds` does), because grabbing the file's first match
     sends an MCP token to the endpoint, earns a 401, and reports a signed-in host as signed out.
     Caught only by running the command against a REAL credentials file, which is why
     `remote-claude-usage.test.ts` runs the generated script under a real `/bin/sh` against a fake
     `$HOME` + fake `curl`, the same discipline as the canvas-control shim.
  3. **A read that could not run is `error`, never `unavailable`**, a dead master says nothing
     about whether the account has a subscription, and 'unavailable' silently drops the row.
  4. **A failed read keeps the last good numbers** (`holdLastGood`, local AND remote): the
     endpoint answers **429** on a budget every Claude CLI using the same login also spends
     (measured on a host running 54 `claude` processes: 429 for minutes on end), and replacing
     the bars with "Could not read usage" made rows flicker for no change in the account. The
     held snapshot is `status: 'error'` + the old limits + the OLD `updatedAt`, so the debounce
     must key on the READ's time (`lastFetchAt`/`remoteCache.at`), never on `updatedAt`, keying
     it on the numbers' age re-reads on every call and hammers the endpoint that said 429.
  Shape: `remoteUsageTargets` (pure) elects ONE connected project per host (several projects share
  a host's `$HOME`) and offers its system `~/.claude` plus every managed account pinned to that
  host. The service (`usage:remote`) caches per target under the usual debounce, evicts targets
  whose host disconnected, and coalesces concurrent reads. **On demand, never polled**, each row
  is an ssh exec plus an HTTPS request on someone else's machine; the renderer asks on mount, on
  popover open, on ⟳, and when the active project's connection comes up (an SSH project is opened
  before its master is ready). Deps are injected exactly like
  Context Link's (`src/main` supplies the ControlMaster; **Server Edition passes none** ⇒ `[]`, so
  the UI needs no capability check). Own Settings switch (`claude-remote`), because hiding local
  Claude usage must not silently take the hosts down with it. **Mobile: N/A**, the
  slice pushed to a host still drops `usage` (a host reading its own numbers back off us is
  pointless), and no keychain leg exists remotely (a headless macOS host would hang on the prompt,
  so a mac host reports nothing).


- **Codex SSH usage** (`core/usage/remote-codex-usage.ts`) uses host-side Node JSON parsing
  and curl, reusing the local Codex quota mapper. Read only `tokens.access_token` and
  `tokens.account_id`; pass headers through stdin, disable curl config/redirects, and return
  only sanitized quota fields. Never download credentials, refresh tokens, or launch a remote
  app-server just to refresh usage. The host needs Node and curl; missing tools/transport or
  malformed responses are errors, not proof of a logged-out account. System reads use the
  host login environment's `CODEX_HOME`; managed reads use the validated account's remote
  home. Cache identity includes provider, host, account and connection/home identity.
  Remote Codex rows obey the Codex visibility setting and carry no Claude default-account
  or bulk-move actions. Desktop supports SSH reads; Server Edition keeps its local core
  readers and absent SSH dependency. The private mobile reader is separate. Device checks:
  [Codex SSH metrics](codex-ssh-metrics.md).


## Rules for contributors

- Claude usage identity is scoped to the same config directory as its credentials. Read organization metadata even when credentials already include an email, and degrade to the email-only row when metadata cannot be read. Managed usage must never fall back to an unscoped system Keychain token: that would label one account's limits with another account's organization.
- Bottom canvas pills must leave room for the measured dock. `CanvasPills` bounds their row and moves it above the dock when the side budget is too small. Only usage summary text may truncate; keep refresh outside that overflow and keep the row free of stacking contexts so popovers can still clear the sidebar and board. `scripts/usage-layout.test.ts` verifies real Chrome layout and hit targets (set `CHROME_BIN` when Chrome is not at the Linux default path).
- Usage readouts distinguish failed reads from empty data. For Claude, show the failure when `status` is `error` and limits are empty, including beside other providers; preserve last-known bars when limits remain. Keep both single-account and multi-account views covered.
- Grok billing diagnostics must keep HTTP codes and safe failure categories per billing view. Never send raw error messages, URLs or response bodies to the UI; credentials remain read-only. A failed view is not proof that there is no quota, even when the other view responds.
- SSH Codex metrics must stay scoped to the host and account being read. Credentials and quota HTTP requests remain on that host; only sanitized usage returns over SSH. Never route a failed remote context lookup into a local transcript reader, or reuse Claude's token formula or window estimate for Codex. The system account follows the host login environment; managed accounts use their validated private home. Usage refresh must not start or repair a shared Codex daemon.
