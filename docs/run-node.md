# Run node

The run node is a `terminal` node with `data.runConfig`, providing a bounded interpretation of `.vscode/launch.json` and Flutter launch configurations. This document covers platform gating, launcher safety, run lifecycle, browser and simulator previews, and the surfaces that support it.

## Windows platform gate

Run configurations (launch.json run node) are gated off for process launches on a Windows core because the launcher is a POSIX `sh` script stopped and hot-reloaded by POSIX signals. The core's
  `gateEntriesForPlatform` marks those entries unsupported with `RUN_WINDOWS_REASON` (browser
  configs and browser-only compounds still run), and the add menu / Dock disable the row (the palette hides it)
  from the core's own platform (`useCoreIsWindows`). A pwsh launcher is the follow-up.

## Run node

**Run node** (a `terminal` node carrying `data.runConfig`, NOT a new NodeKind;
`@shared/run-config`, `core/run-service.ts`, `nodes/RunBar.tsx`), VS Code's "Run Without
Debugging" for any `.vscode/launch.json` configuration. The toolbar picks a folder (defaults to
the project's; sibling and `*.worktrees/*` folders with a launch.json or a Flutter pubspec are
offered), a configuration or compound (stored by NAME, re-read at every run) and, for Flutter, a
device. **launch.json runs nothing by itself**, each `type` belongs to an extension, so
`planLaunch` is the table of what each extension would run: dart (flutter run / flutter test /
dart run / dart test), node/pwa-node/node-terminal, python/debugpy, go, php, coreclr,
lldb/cppdbg, and chrome/msedge (a browser node). Everything else is REFUSED with a sentence,
never guessed: debugging itself, `"request": "attach"`, unknown types, PHP "listen for Xdebug",
and variables that need an editor or extension (`${file}`, `${input:}`, `${command:}` except the
Python interpreter one, `${config:}`). `preLaunchTask` (+ `dependsOn`, `${defaultBuildTask}`)
runs shell/process/npm/dart/flutter/typescript/cargo tasks first; a background (watch) task is
refused. Compounds run their first member in the node and open the rest beside it, started.
Rules a refactor must keep: (1) **nothing from a repo is ever typed into a shell**, the host
writes a POSIX launcher (`buildLauncher`, every token single-quoted, owner-only via
`writeFileAtomic`) and only `sh '<launcher>'` is typed; env and `envFile` values live in that
file, never in shell history; tested against injection and run for real under `/bin/sh`;
(2) the launcher records its pid and exit status, and traps INT/TERM with a HANDLER (`trap ':'`,
never `''`, which children would inherit as "ignore"); **Stop is SIGINT to the launcher's process
group** (Ctrl+C), SIGTERM if it will not go, and only the launcher + its children when it is not
a group leader, never a group that could hold the user's shell; (3) a pid is signalled only
while `ps` says it is still ours (the launcher's path, or flutter for hot reload/restart, which
are SIGUSR1/SIGUSR2 to `flutter run --pid-file`); (4) Flutter reload on save is a core
`fs.watch` of `<dir>/lib` that deliberately OUTLIVES the node's view; (5) the terminal is hidden
by default (`runConfig.showTerminal`, toggled by ⋯ with the extra-args field): collapse's
`display: none` path via `.term-node:has(.run-bar--compact)` and a node height fitted to the
rows; a run that ends without our Stop says where to look; (6) **switching folders keeps the app
when it can** (`canHotSwitch`: a running Flutter run, same device, same flavor, Flutter on both
sides): the toolbar offers **Switch (keep app)** beside **Rebuild**, type `d` (flutter's own
detach: the tool exits, the app keeps running), `flutter attach` from the new folder (only the
flags attach accepts survive, `attachArgs`, MEASURED against `flutter attach --help`, attach has
no `--flavor`), wait until it prints its key help AFTER its own `▶ flutter attach` line
(`attachConnected`), then a hot RESTART, never a reload: a freshly attached tool only pushes
files changed after it connected (MEASURED live, Flutter 3.47 / iOS 27.1 simulator: a reload
after attaching from the other checkout "Reloaded 0 libraries" and the old code kept running; a
SIGUSR2 restart swapped it in 2.6 s, same app pid, no rebuild). **On an iOS simulator attach
needs `--debug-url`**: it finds an already-running app by mDNS, which the simulator barely
supports ("The Dart VM Service was not discovered after 30 seconds"), and the URL `flutter run`
printed is its DDS proxy, gone with the detached tool. `simulatorVmServiceUrl` reads the app's
own "The Dart VM service is listening on …" line from `simctl spawn <udid> log show` (~3 s) and
takes the newest one whose process is still alive (`parseVmServiceLog`); not found ⇒ the switch
is refused with Rebuild offered. Some Macs ship no registered `Simulator.app` (`open -a
Simulator` fails); booting falls back to the active Xcode's copy and otherwise runs headless. Flutter REFUSES a hand-passed `FLUTTER_APP_FLAVOR`, so after an
attach `appFlavor` is the pubspec default; an app whose `lib/` reads `appFlavor` is refused the
switch with that reason. Dart `"request": "attach"` configurations run as `flutter attach` too;
every other attach is still refused. No "bring simulator forward":
Simulator.app can only be activated as a whole (same-named simulators are told apart by id in
the dropdown). Add menus: **New view ▸ New run configuration**. Local projects only (disabled
with `RUN_SSH_HINT` in SSH projects; relay stub answers "managed on the host"); POSIX only
(refused on Windows). Server Edition: real. Kanban card modal: not yet. Mobile: N/A.
