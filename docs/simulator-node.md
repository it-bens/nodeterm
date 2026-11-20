# Simulator node

The `simulator` node is a live iOS or Android device screen on the canvas. It covers the iOS bridge and its private-framework boundary, Android's emulator gRPC transport, input, device actions, and the shared inline preview used by run nodes. Desktop is full; Server Edition and relay tabs refuse the node, and mobile has no canvas surface.

**simulator** (`SimulatorNode.tsx`, `core/simulator/`, `@shared/simulator`), a live iOS simulator
screen on the canvas: click = tap, drag = swipe, wheel = a swipe, typing goes to the device (HID
keyboard usages from `KeyboardEvent.code`; ⌘-chords stay with nodeterm), Home / Lock buttons, a
device picker with Boot, and a Cover/Inner screen choice on foldables. DeviceHub (Xcode 27's
replacement for Simulator.app) cannot be embedded, it exposes only a `devices://` URL scheme and
macOS has no cross-app window embedding, so the node draws the device itself through
`nt-simbridge`, a small Swift helper whose SOURCE ships in `simbridge-source.ts` and is compiled on
first use with `xcrun swiftc` into `<userData>/simulator-bridge/<hash>/` (hash = source + version +
Xcode build, so an Xcode update recompiles; MEASURED ~3 s). It uses Xcode's PRIVATE simulator
frameworks (CoreSimulator / SimulatorKit, what DeviceHub uses), resolved at RUNTIME (dlopen /
NSClassFromString / dlsym) so a moved symbol fails with a sentence, never a link-time crash; the
Indigo HID message layout follows facebook/idb (MIT). Rules a refactor must keep, all MEASURED on
Xcode 27: (1) frames are the display's IOSurface, encoded only when its seed changes (0 cost while
still; ~19–30 fps while moving; first frame ~0.4 s); (2) a device can have SEVERAL main-class
displays, a foldable ("iPhone Duo") has an inner and a cover screen and only one is lit, so the
helper shows the lit one and follows a fold (switches when the shown one is still and another just
changed), with a manual pin; (3) touches go to the SHOWN screen's own digitizer, `0x40000000 |
screenID` (`screenProperties.screenID`): the generic main-screen target 0x32 reached neither screen
on the foldable; (4) Home/Lock are HID Consumer usages (Menu 0x40 / Power 0x30) to 0x32, a Face ID
device ignores the legacy home-button source, falling back to the legacy source (0x33) when
refused; (5) the display state objects are ROCK proxies: no key-value coding, message primitives
through typed `objc_msgSend`. Input from the renderer is re-validated (`normalizeSimulatorInput`)
before it reaches the helper. A run node with a simulator device shows a 📱 button that opens (or
focuses) the Simulator node for it. **Desktop only** (iOS devices need macOS; Android ones run
everywhere): the add rows are disabled with `SIMULATOR_UNAVAILABLE_HINT` in a browser tab and
with `SIMULATOR_RELAY_HINT` in a relay tab (`simulatorRefusal`, one rule for the menu, the Dock,
the palette and `addSimulator`), and a Simulator node that reaches a relay tab anyway (it is in
the host's project file) shows that reason instead of streaming, a relay tab's project is the
host's, and the devices this renderer can reach are THIS machine's. `simulator:*` is also in
`HOST_ONLY_CHANNEL_PREFIXES`, so a relay peer is refused host-side. Kanban: not a card. Mobile: N/A.
**A device takes input from its FIRST HID client only** (MEASURED: a second client's touches are
dropped without an error). So the host runs ONE helper per device, shared by every node showing
it (`bridges` keyed by UDID, a late joiner replayed the last displays/ready). That is why the
Simulator node's OWN Boot is headless (`bootSimulator(id, { openWindow: false })`): a window
showing the device is a client too, and it took touch input away from the node. A run node's
Boot (`run:boot-device`) still opens Simulator.app as it always has, a user with no Simulator
node must still see the device. "Open in DeviceHub" (⋯ menu) says so in its tooltip.
**The ⋯ menu** (`simulatorMenu.tsx`, pure) is a complete map of DeviceHub's menus, from the
`…MenuItemProvider` names in Xcode 27's DeviceKit: what nodeterm can do runs, HID buttons
(Home/Lock/Side/Siri/Volume/Play-Pause) and rotation through the helper; everything else through
public `simctl` in `core/simulator/simulator-actions.ts` (appearance, text size, Increase
Contrast, location set/scenario/clear, status-bar overrides, shake and Face/Touch ID via the
simulator's own `notifyutil` notifications, Open URL, push, privacy, pasteboard sync, install,
add media, screenshot/recording, restart, erase), and what it cannot is listed greyed out with
"(Not Supported)" and the reason (App Switcher: the edge-swipe gesture is not recognised from
outside DeviceHub; Action Button; Grayscale/Reduce Motion/Reduce Transparency/Liquid Glass: no
public command; tvOS/watchOS/visionOS controls; device management). Actions are re-validated
(`normalizeSimulatorAction`) and every argument is its own argv entry. All the simctl actions
were run live against an iOS 27 simulator; `simctl location list` is a TABLE
(`parseLocationScenarios` reads the first column).
**Edge swipes carry the panel edge they start on** (`touchEdge`, `TOUCH_EDGE_MARGIN` 2.5%):
iOS recognises its system edge gestures, the home swipe up from the bottom, from the Indigo
contact's edge field, not from where the finger lands. MEASURED (Xcode 27): without the flag a
swipe up from the very bottom only scrolled the app; 1 top / 2 left / 3 bottom / 4 right name
PHYSICAL panel edges (an iPad turned left went home with 2, turned right with 4; top inferred), so
the node computes the edge after mapping the touch to the panel and it holds for the whole
contact. A swipe down from the top worked without it.
**Inside a run node.** The screen, toolbar, ⋯ menu and input are one component, `SimulatorView`;
`SimulatorNode` is a thin box around it (title, close, resize handles, fit-to-screen), and a run
node mounts the same view under its control rows when `data.runSimulator` is present (persisted,
`normalizeInlineSimulatorConfig`: the device only, the panel FILLS the node, so resizing the
node scales the screen, letterboxed; no size of its own is stored). 📱 on every run node toggles that panel
(the run's device preselected when it is an iOS UDID or an Android emulator's adb serial,
`SimulatorDevice.serial`; else empty to pick); ⇱ splits it into a Simulator node carrying
`dockTo: <runNodeId>`, whose ⇲ (or the run node's 📱) docks it back. The inline stream is keyed
`<runNodeId>.sim`. While it is open the compact (terminal hidden) height fit stands down, the
node's size is the user's, and the run bar flexes to fill the node; opening grows the node by
`INLINE_SIM_HEIGHT`, closing hands that back. Inline there is no node of its own to snap to the
screen's shape, so Actual Size / Fit to Screen are greyed out there (pop out for those).
**The browser panel** (`@shared/run-preview`) is the same idea for a run whose result is a page:
there is NO choice, a node runs one thing, so the button shows the panel the run calls for (📱
or 🌐, `previewKindFor`), and an OPEN panel follows the run (switch the device to Chrome and the
simulator becomes the browser, and back), a `chrome`/`msedge` launch configuration,
a Flutter web device, picked OR named by the configuration itself (`-d chrome` in its args, a
rule that read only the picker showed 📱 for exactly those runs), or a run with no device, i.e.
a dev server). `data.runBrowser` (persisted, http(s) only) holds the page: a browser
configuration's own `url` (its Run/Open now lands in this panel, or in a panel popped out of it,
instead of a separate browser node), else the URL picked out of the run's own output by
`localUrlFromOutput` (the newest local http(s) URL on a "served at / Local: / listening on" line;
DevTools / VM-service lines skipped; `0.0.0.0` opened as `localhost`), polled with `pty.capture`
only while the panel is open, the run is up and no page is known. A picked-out URL is marked
`auto` and re-read on the next run (the port may change); one the person navigated to is kept.
A Flutter `chrome`/`edge` run is EMBEDDED only when the configuration opts in
(`runConfig.embedBrowser: true`, ⋯ → "Show web inside this node", shown for a Flutter web run;
`embedsFlutterWeb` is the one rule for host and bar), an existing configuration keeps Flutter's
own Chrome window. Opted in, `planLaunch` swaps the device for
`web-server` (`webServerDevice`, `PlanContext.webServer`) wherever it is named, the pick, `-d` /
`--device-id` / `-d=` in args or toolArgs, or Dart-Code's `deviceId` field (which used to be
dropped, not passed as `-d`), because Chrome would open a window of Flutter's own while the web
server serves the same app to any browser and prints its URL; Run then opens the browser panel
by itself. ⇱ pops it into a browser node with `dockTo`; its ⇲
(`nodeterm:dock-preview`, shared with the Simulator node) or 🌐 docks it back. A run node holds
one panel at a time. The inline page is not part of the browser-node keep-alive pool, so a
project switch reloads it.
**Android virtual devices** use the same node (`core/simulator/android-*.ts`). A device id is
`avd:<name>` (the AVD name is stable; adb serials are not) in the same persisted `udid` field.
The picker lists every AVD read from files (`<avdHome>/<name>.ini` → `config.ini`; no
avdmanager, which needs a JDK), booted ones from the emulator's discovery files
(`…/avd/running/pid_<pid>.ini`, pid checked: a crash leaves the file). Boot runs
`emulator -avd <name> -no-window`, headless like iOS, logging to `<userData>/android-emulator/`.
Everything streams over the emulator's own gRPC interface, spoken with a hand-written protobuf
codec over `node:http2` (`android-grpc.ts`, no dependency), authenticated with the per-run
`grpc.token` from the discovery file. MEASURED (emulator 36.6.11): PNG frames fitted into a
SQUARE box (one box serves both orientations; width alone is ignored) run ~25 fps while moving
and send nothing while still; keys need the full USB usage (`0x70000 | usage`, a bare usage
types nothing); Home/Back/Recents/Power are W3C key names (`GoHome`, `GoBack`, `AppSwitch`,
`Power`); rotation is `setPhysicalModel` ROTATION z (90 = turned left) and frames then arrive
ALREADY TURNED while touches stay in PORTRAIT panel pixels, so the node does not turn an Android
picture (`picturePreRotated`) but maps touches with the same `displayToFramebuffer`. A rotation
of a portrait-locked app changes no guest pixels, so the stream sends nothing; the session takes
a `getScreenshot` after each rotation. Input is queued and sent one call at a time (drag points
coalesced) so it reaches the emulator in order. ⌘V sets the device clipboard (`setClipboard`)
and presses Ctrl+V. Menu actions use gRPC where it exists (battery, GPS, clipboard, fingerprint,
restart, shutdown, screenshots) and `adb -s emulator-<port>` otherwise (night mode, font scale,
demo-mode status bar, URLs, APK installs, media), every `adb shell` argument single-quoted for
the DEVICE's shell. Not yet: screen recording, wipe data, foldables, the run node's 📱 for an
Android emulator. The node is offered in the desktop app on every OS (iOS devices are simply
absent off macOS); the Server Edition and relay tabs still refuse it. Install / Add Media accept a
POSIX or a Windows drive path (`C:\…`, `C:/…`; UNC refused), since the emulator runs on Windows too.
The emulator's discovery file is trusted only when every directory from the temp/runtime/home
base down and the file itself are ours (uid, no symlink, file opened `O_NOFOLLOW`), and a pid we
may not signal (EPERM) is another user's, not our emulator, `/tmp/android-<user>` can be created
by anyone first, and a planted `grpc.port` would receive every key typed into the device.
