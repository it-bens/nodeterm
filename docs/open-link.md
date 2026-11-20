# `nodeterm://` links

`nodeterm://open?agent=...&project=...|project-cwd=...&cwd=...&prompt=...|prompt-file=...&title=...&source=...` lets an app with no node identity (a review tool, a script, a bookmark) ask for one node. The human approves every link in a confirm sheet. A link proves nothing about who sent it (any web page can build one), so the confirm sheet is its whole authority.

Spec and as-built decisions: [open-link design](superpowers/specs/2026-10-13-open-link-design.md). Code: parser `@shared/open-link`, service `src/main/open-link.ts` (electron-free, tested), wiring `src/main/open-link-wiring.ts`, sheet `src/renderer/components/OpenLinkSheet.tsx`.

## Invariants

- **Behind a flag, off by default** (`settings.allowOpenLinks`, Settings → Behavior "Allow nodeterm:// links"). Main reads it on every link and again on accept; main checks the flag itself, not the renderer. Off, a link is not even parsed: one notice says where to turn it on. The scheme is still registered, so that notice is what a click meets.
- **No power flags.** A link opens one node in the project's default permission mode. Never add a param for a permission mode, `--after`, a model, a count or "open without asking". The signed-link design in the spec is the only planned way to skip the sheet. `source` is display-only.
- **The parser refuses, never drops:** unknown or duplicate params, over-cap values, relative paths, controls.
- **The renderer only chooses.** Its answer is a candidate id main offered, "new project", and a title. The cwd and prompt in the final open come from main, never the reply.
- **Paths are stat'ed before the sheet, so a path may not reach another machine.** A POSIX-form path with a backslash is refused (Windows reads `/\host\share` as UNC, and a stat there authenticates to that host over NTLM), as are UNC paths and the macOS automount roots `/net` and `/Network`.
- **Nothing the sheet would not show reaches the agent.** `hasHiddenText` (every `\p{Cf}` except ZWJ, plus C1: bidi overrides, zero-width, Unicode TAG "ASCII smuggling") refuses an inline prompt, a prompt file's text, every path and every label. The sheet prints the prompt's size.
- **The permission mode shown is the one launched.** A new-project choice adopts the folder's committed `project.json`, so the sheet probes it (Open waits) and labels from it; a mode that differs at landing refuses. `bypassPermissions` gets a warning in the sheet.
- **The prompt file is read from one non-blocking open** (no stat first, `O_NONBLOCK` so a FIFO cannot park a thread, looped read to EOF or the cap). A project whose folder changed while the sheet was up refuses on accept. `/`, the home folder and its ancestors are never offered as a new project.
- **Main is the authority.** It resolves project candidates from its own workspace index (SSH and handed-off projects are never offered), stats the cwd, reads `prompt-file`, and on accept re-checks the renderer's choice (a candidate id or the offered new-project folder), re-reads the file, refuses if it changed, and copies it into `<userData>/launch-prompts`. Requests are single use and expire after 10 minutes.
- **Delivery.** `open-url` is registered in `will-finish-launching` (macOS); on Windows and Linux the link arrives in argv on cold start and in `second-instance`. Links are held until the renderer calls `openLink.ready()`, and held again on every page death (`clearRendererKeyState`). Limit: 5 links a minute. A second link replaces only an untouched sheet. The window raise is the shared `raiseForUserLaunch`.
- **The sheet.** Enter never opens it, nor does Space on the focused button (`detail === 0` is ignored). Open is disabled for 1 s, then accepts a pointer click only. A second link replaces the sheet only within that first second and untouched; after that it is dropped (reading fires no event). An accepted link waiting for its project to load is dropped after 30 s, never created later.
- `pair` and `join` links open the existing code prompt pre-filled; they never auto-connect.

## Surfaces

Desktop only. The Server Edition stub is inert, and `open-link:` is a host-only prefix. A dev run registers the scheme only with `NODETERM_DEV_PROTOCOL=1`. An `NT_MULTI` sandbox takes links (argv, `second-instance`, `open -a <Electron.app> 'nodeterm://...'`) only with that variable and never registers the scheme; that is the live-test setup beside an installed app.
