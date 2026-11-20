# Agent-integration consent

Issue #744 (redo of the superseded draft #900). nodeterm used to write into every agent CLI's
user-owned global configuration on every launch, and onto every SSH host at connect, with no
recorded decision and no way to say no: status hooks in `~/.claude/settings.json`,
`~/.gemini/settings.json`, `~/.codex/hooks.json` + `config.toml` trust, grok/copilot/opencode/
antigravity hook files, and ~17 KB of canvas-control + context-link instructions merged into
`~/.codex/AGENTS.md`, `~/.gemini/GEMINI.md`, copilot's and opencode's instruction files. The codex
blocks alone took 17,102 bytes of codex's 32 KiB `project_doc_max_bytes` budget, crowding out the
user's own repository instructions.

## The model

`settings.agentIntegrations` (`src/shared/agent-integrations.ts`), hand-editable and sanitized on
every read:

| field | meaning |
|---|---|
| `agents[<id>]` | `enabled` / `declined` per agent on THIS machine; absent = never asked |
| `hosts[<user@host>]` | per SSH host; keyed on `sshHostKey` only, so editing the identity file, port or extra args keeps the answer |
| `hostDefault` | the answer for a host with no entry; set to `enabled` only by grandfathering |
| `origin` | `grandfathered` (an install from before this feature) or `asked` |
| `noticeDismissed` | the grandfathered install's one-time notice was dismissed |

Per agent the lifecycle (`src/core/agent-integrations.ts`, booted by BOTH shells) does:

- **enabled**: install/refresh the status hook and the two skills (canvas control, context link) in
  that agent's OWN skills dir; strip the legacy marker blocks older builds merged into its global
  instruction file.
- **declined**: remove exactly what nodeterm wrote — hook entries by exact command match (a user's
  own hooks survive), skill files by exact-content receipt (a file the user edited is kept and
  listed in Settings), the legacy blocks, our hook scripts.
- **undecided**: nothing, in either direction.

The Server Edition's `installHooks: false` is a hard veto that outranks every choice.

## Default for existing vs new installs (product decision — confirm)

- **Existing installs are grandfathered as enabled.** Evidence is our own files only: a non-empty
  `~/.nodeterm/agent-hooks/`. The canvas/context shims under userData are NOT evidence — every boot
  writes them, consent or not, which grandfathered fresh installs on their second boot (#1149). The
  decision is taken once: the first boot without evidence writes
  `<userData>/integration-consent-fresh-install`, and no later boot of that install grandfathers
  (not even when a sibling install sharing `~/.nodeterm` enabled its own hooks). The record is written at
  boot, before the renderer loads settings and before any install, with every agent enabled and
  `hostDefault: enabled`, so nobody's badges or canvas control stop on upgrade. A one-time
  dismissible notice says integrations are now a choice and links to Settings → Agents.
- **New installs are asked once**, by a banner (Enable all / Choose… / Not now). Until answered
  nothing global is written; terminals work, but agents have no status badges, notifications,
  session names, context meter, subagent cards, canvas control or linked-node reads. "Not now"
  records a decline for every agent, so it is asked once.
- **SSH hosts**: a host with no answer gets nothing (and nothing is removed). When the active
  project's host is connected and unanswered, a banner asks (Allow on this host / Not on this host).
  A host installs an agent only when BOTH the host and that agent are enabled.

## Where the skills go (measured 2026-10-03)

Local, each agent's own dir (so a per-agent decline is precise):

| agent | dir | measured with |
|---|---|---|
| claude | `$CLAUDE_CONFIG_DIR` or `~/.claude` `/skills`, plus every local managed/linked account dir | (existing) |
| codex 0.156.1 | `$CODEX_HOME/skills` (+ managed codex homes) | `codex debug prompt-input` |
| gemini 0.62.0 | `$GEMINI_CLI_HOME`/`~` `/.gemini/skills` | `gemini skills list --all` |
| opencode 1.18.33 | `<opencode config>/skills` | `opencode debug skill` |
| copilot 1.0.89 | `$COPILOT_HOME`/`~/.copilot` `/skills` | `copilot skill --help` (documented) |
| grok 1.0.44 | `$GROK_HOME`/`~/.grok` `/skills` | `grok inspect --json` |

On an SSH host: claude's `~/.claude/skills` (and remote account dirs), and **`~/.agents/skills`**
for everyone else — codex, gemini, opencode and grok were all measured reading it, copilot documents
it — because one env-independent dir is what a host we cannot introspect cheaply needs.

Copilot has no context link, so it gets the canvas skill only (as before). Antigravity has neither.

## What is deliberately not done

- `config.toml` codex trust entries are left on decline: a trust entry for a hook that no longer
  exists is inert, and the file is the user's.
- The `buildCanvasControlInstructions` / `buildLinkedContextInstructions` builders are no longer
  installed anywhere; their parity tests still pin them and should move to the skill bodies
  (follow-up).
- A file that held ONLY our block is removed locally; on an SSH host (or behind a symlink) it is
  left with one newline, because the guarded transaction never publishes an empty file.
- An SSH host's removal only recognises skill files this build would write, or whose bytes were
  recorded when this machine wrote them (receipts in `<userData>/integration-receipts.json`); an
  older build's copy on a host this machine never refreshed is kept and reported.

## Surfaces

Desktop: full. Server Edition: the same lifecycle and Settings rows (`installHooks: false` vetoes),
no SSH hosts. Relay tabs: the consent is the host's (settings are host-only); the relay tab's own
Settings describe THIS machine. Mobile: N/A for v1 — the phone reads status the host's hooks
produce; a phone-side explanation of "integration off" is an iOS follow-up.

## Device checklist (not runnable here)

1. macOS desktop upgrade from 0.4.x: grandfathered notice appears once; badges keep working.
2. Fresh macOS install: banner appears before any `~/.claude` write (`ls ~/.claude` unchanged);
   Enable all → badges and canvas control work.
3. Decline claude on macOS: `~/.claude/settings.json` keeps the user's own hooks, our entries gone;
   an edited `SKILL.md` is listed in Settings.
4. Codex on macOS reads the canvas skill from `~/.codex/skills` (ask it to list nodes).
5. SSH host (Linux, codex + gemini + claude installed): Allow → `~/.agents/skills` and
   `~/.claude/skills` populated, no `AGENTS.md` block; Not on this host → cleaned.
6. SSH host with an old 17 KB `AGENTS.md` block: stripped on the first connect after upgrade.
7. Copilot on a real login: `copilot skill list` shows manage-nodeterm-canvas from `~/.copilot/skills`
   (local) and `~/.agents/skills` (SSH) — only the help text was measured here.
8. Windows desktop: hooks for a consented codex (`codex-hook.cmd`) and the skills dirs resolve.

## Rules a change must keep

Additions to the model above (the grandfathering evidence rule and the once-only marker are already described in "Default for existing vs new installs"; the points below are the ones not stated there).

- **Nothing writes into an agent's global config except `src/core/agent-integrations.ts`** (issue #744). Hooks, skills and anything else under `~/.claude`, `~/.codex`, `~/.gemini`, `~/.grok`, `~/.copilot`, opencode's config dir and `~/.agents`, locally or on an SSH host, are installed only for an agent the user enabled in `settings.agentIntegrations`, removed when they decline, and left alone while they have not answered. Do not add a direct call to an installer: add the artifact to the lifecycle (local) or to the `RemoteHooks` artifact plan (SSH), and give it a removal that deletes only what nodeterm wrote. Do not merge text into `AGENTS.md` or `GEMINI.md`: agents get skills (progressive disclosure) instead; the old blocks were 17 KB of codex's 32 KiB `project_doc_max_bytes`.
- **Grandfathering is decided at boot, before `settingsStore.registerIpc()` and before any install.** The detection (`detectExistingIntegrations`) must use a file only a pre-consent build wrote. A file nodeterm writes at every boot is never evidence of a pre-consent install: counting the always-written userData shims grandfathered every fresh install on its second boot (issue #1149: a systemd restart, the update timer, or a desktop quit before the banner was answered, leaving 11 hook entries in `~/.claude/settings.json` and Codex `trusted_hash` pre-approvals). The once-only decision is the `FRESH_INSTALL_MARKER` file (`<userData>/integration-consent-fresh-install`), not a settings key: `SettingsStore.init` reads an unreadable `settings.json` as the defaults, and a boot-time save of a "not answered yet" record would overwrite the user's file. Same function, same call, both shells.
- **No caller reaches an installer by omission.** `installManagedAgentHooks(agents)` takes a required set; `RemoteHooks` takes a required plan (`NO_REMOTE_INTEGRATION` is fail-closed); account add and link ask `currentIntegrationLifecycle()` and write nothing when none is registered. `agent-integrations-wiring.test.ts` pins every call site of the global installers and fails on a new one.
- **SSH consent is per host** (`sshHostKey`, never identity file, port or args). A host installs an agent only when the host and the agent are enabled. An unanswered host is not touched at all (no clean-up, no ssh). Removals run on connect and when the plan changes (`SshProjectManager.onIntegrationConsentChanged`). A removal never creates a missing config file (`stripRemoteSettingsFile`).
- **`$CLAUDE_CONFIG_DIR` decides the system Claude dir** (`claudeSystemConfigDir`), never a hardcoded `~/.claude`; a decline also cleans the legacy `~/.claude` when they differ.
- **The fullscreen-TUI write is part of the claude integration** and follows its consent (see [hook model](hook-model.md)).
- The Codex hook-trust repair at spawn is consent-gated too: it runs only through `currentIntegrationLifecycle()?.ensureCodexHooksForLaunch(home)`, a no-op unless codex is enabled (see [hook model](hook-model.md)).
