# Speech and dictation

Voice-to-text input captured through the microphone and turned into terminal text by on-device Whisper. It works on desktop (Electron) and the Server Edition (browser). iOS support is separate (`nodeterm-ios`, a private repo). Code: `src/core/speech/`, `src/shared/speech.ts`, the renderer capture and overlay, and `scripts/patch-smart-whisper.mjs`.

## Service seam and IPC

- `SpeechService` (core) plus the `PlatformSpeechProvider` interface and the shell implementations (`PlatformElectron` / `PlatformServer`). Models are stored under `${dataDir}/speech-models/`, with fenced downloads and an orphan sweep (`removeUnusedModels`).
- Core validates the license: **tiny** is free (always); **base, small and large-v3-turbo** are Pro (via `isPremium()`). One model is loaded at a time (FIFO memory management). The smart-whisper import is lazy and degrades to a friendly error if the native dependency is unavailable (`"Local whisper is unavailable..."`).
- **Cloud contract (iOS parity):** a `/v1/transcribe` multipart endpoint, not built yet (the SDK `transcribe()` call matches iOS byte for byte), for future remote transcription.
- IPC channels `speech:*` (in `src/shared/ipc.ts`) are wired in both Electron and Server: `speech:transcribe` (returns `Promise<{text}>`), `speech:models`, `speech:model-download`, `speech:model-delete`, `speech:progress` (main or server to renderer download-progress broadcast), and `speech:mic-consent` (Electron mic prompt only; the server always answers true). There is no `speech:synthesize` or `speech:cancel`, and no audio in the reply.

## Renderer capture

`PcmCapture` is an AudioWorklet (16 kHz single-channel PCM, WebAudio or a fallback ScriptProcessor) feeding the DictationOverlay (dock mic or ⌘⇧D, or the Cmd key; Settings → Speech has model choice and progress). **Send** appends text plus Enter to the terminal; **Insert** sends text only via `sendText(..., {enter: false})`. Nothing auto-submits: the user always decides when to send.

## Language

`SPEECH_LANGUAGES` (`src/shared/speech.ts`) is whisper's own `LANGUAGES` table (tokenizer.py) verbatim: 100 entries carrying the code, CLDR's English name, the endonym and the alternate spellings people type (whisper's own name where it differs, plus its documented alias table). Cantonese is flagged `sinceV3`, the one entry the pre-large-v3 models have no token for.

It replaced a 7-entry array inside `SpeechSection`, which was the only limit in the whole stack (issue #586). `SpeechSettings.language` is a free string nothing validates on the way to disk, and whisper.cpp takes any code, so `"language": "pl"` hand-edited into settings.json transcribed Polish correctly while the dropdown rendered blank and overwrote it on the next click in the row. Three rules come out of that:

- The control is the app's searchable menu idiom (`SpeechLanguageSelect`: `.bind-select` trigger plus a portaled `.tab-menu` with a pinned filter over a scrolling list, the Source Control branch quick-pick's shape), never a `<select>`. 101 rows with no search, unreachable by typing "polski", is not a picker. Rows are the pure `renderer/lib/speechLanguageRows.ts`.
- **A code we cannot name is still the user's setting.** `speechLanguageLabel` returns an unknown code as-is (not `''`) and the picker gives it its own row, so a display gap can never become data loss the way the `<select>` made it.
- **The cloud `locale` is passed through unchanged, `auto` included.** `register-ipc.ts` used to send `language === 'auto' ? 'en' : language`, so on the Cloud engine "Auto-detect" was a hard, silent English, on the one engine where a missing language could not be worked around at all. `/v1/transcribe` does not exist yet, so `auto` meaning detect is our contract to write.

Deliberately not done: seeding the initial value from the system locale (issue #586 §3). The default is still `auto`. Mobile keeps its own list, tracked separately as issue #591.

## Browser constraints

`getUserMedia` requires HTTPS or `localhost`. The mic permission prompt is the browser's own (nodeterm does not handle it). Model downloads land in the server's data dir (accessible across sessions).

## Electron and the native dependency

smart-whisper is externalized and `asarUnpack`'d (not bundled); `postinstall` rebuilds it against Electron's ABI. Device verification of the ABI rebuild has not been exercised on a dev machine: test paths exist but have not been run in CI.

`scripts/patch-smart-whisper.mjs` carries three patches, each tied to a measurement.

### Linux x64 ships two whisper binaries, picked per CPU (issue #1267)

smart-whisper's binding.gyp passes no `-m` flags, so on x86-64 ggml compiles at the SSE2 baseline and every AVX/AVX2/FMA/F16C kernel drops out (`objdump -d smart-whisper.node | grep -c ymm` is 0 on the shipped build). Measured, base model, 3 s of audio, 12-thread AVX2 box: 18.8 s as shipped vs 1.9 s with `-mavx -mavx2 -mfma -mf16c`, and the same 11.2 s to 1.5 s under Electron's ABI. The report was large-v3-turbo taking about 7 minutes for one sentence.

- The script (run before `electron-rebuild` in `postinstall`/`rebuild`, beside the node-pty patch) adds a `smart-whisper-avx2` target behind a gyp `OS=="linux" and target_arch=="x64"` condition (mac and Windows builds are unchanged) and makes smart-whisper's loader read `globalThis.__NODETERM_WHISPER_BINARY__`. `pickWhisperBinary` sets it from `/proc/cpuinfo`.
- **Never ship a single `-mavx2` build.** On a CPU without AVX2 that is a SIGILL that takes the whole app down on the first dictation. That is why the baseline stays, and why every failure (flags unreadable, variant file absent, variant fails to load) falls back to it rather than to "unavailable".
- `WHISPER_SIMD_FLAGS` and the script's `SIMD_FLAGS` must name the same set (pinned by `smart-whisper-patch.test.ts`, which also fails on an unpatched install).
- Threads: `whisperThreads` replaces whisper.cpp's `min(4, cpus)` with about half the logical CPUs plus two, capped at 8. Not every CPU: ggml's workers spin-wait, and 12 threads on 12 busy logical CPUs measured 12.8 s against 8 threads' 1.4 s.
- Not covered yet: macOS x64 (Accelerate, same missing SIMD flags in ggml), a GPU backend (this smart-whisper has no Vulkan build option), cancelling a running transcription, and the Server Edition (installed with `--ignore-scripts`, so it keeps the baseline). The startup log line `[speech] whisper binary: ...` says which binary loaded.

### macOS: the Metal shader path points at the unpacked copy

Second patch, marker `NODETERM-PATCH(smart-whisper-metal-asar)`. smart-whisper sets `GGML_METAL_PATH_RESOURCES` from its own `__dirname`, which in the packaged app is inside `app.asar`. Native code cannot read inside an asar ("Not a directory"), so Metal init failed and whisper.cpp ran on the CPU with no error shown. The shader file is already unpacked (asarUnpack), so the patch rewrites `app.asar` to `app.asar.unpacked` in that path. Measured with the app's own binary on an M5 Pro, large-v3-turbo, 2.7 s clip: 5.97 s on the CPU, 0.64 s on Metal; in the app, dictation went from several seconds to under one. A dev install and an explicit `GGML_METAL_PATH_RESOURCES` are unchanged.

### Model params start from whisper.cpp's defaults (every platform)

Third patch, marker `NODETERM-PATCH(smart-whisper-ctx-params)`. smart-whisper's `LoadModel` declared `whisper_context_params params;` and set only `use_gpu`, so DTW, `gpu_device`, `flash_attn` and the rest were uninitialized stack memory and the result depended on what was there. Measured on an M5 Pro, Electron 42, large-v3-turbo: the shipped binary crashed in 4 of 4 runs outside the app (`aheads_masks_init`, then a crash in `whisper_free_state`, because garbage switched DTW on with large-v3's alignment heads). With the defaults it ran without a crash: 0.56 s for a 2.7 s clip and 0.93 s for 21 s on Metal.

The slow dictation inside the packaged app was a separate bug (Metal could not read its shaders inside app.asar, above). A newer whisper.cpp is faster again (0.22 s and 0.43 s, measured the same way) but needs a build change on every platform; that is a separate follow-up.
