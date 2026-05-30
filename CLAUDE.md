# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build

This is an Xcode project; there is no `xcodebuild test` target.

- Debug build (visionOS device, no simulator launch):
  `xcodebuild -project ALVRClient.xcodeproj -scheme ALVRClient -configuration Debug -destination 'generic/platform=visionOS' build`
- After any code change, run that build before declaring work done. Fix failures or report the error summary.

### Rebuilding the ALVR Rust core (rare)

`ALVRClientCore.xcframework` (checked in under `ALVRClient/`) is the pre-built Rust core. You only rebuild it when bumping the `ALVR` git submodule or touching `alvr_client_core`:

1. `git submodule update --init --recursive` (the `ALVR/` dir is empty until you do this)
2. `./build_and_repack.sh` — installs `cbindgen`, adds the `aarch64-apple-ios` Rust target, runs `cargo build -p alvr_client_core --profile distribution`, regenerates `alvr_client_core.h` via cbindgen, then calls `repack_alvr_client.sh`
3. `repack_alvr_client.sh` lipo/vtool's the resulting dylib into a four-slice xcframework (ios, maccatalyst, xros, xrsimulator) and drops it at `ALVRClient/ALVRClientCore.xcframework`

The Swift bridging header `ALVRClient/ALVRClient-Bridging-Header.h` exposes the C API from `alvr_client_core.h`.

## xcconfig layering

`ALVRClient.xcconfig` includes (optionally) `AppStore.xcconfig` and `Override.xcconfig`. `Override.xcconfig` is gitignored — use it for local `DEVELOPMENT_TEAM` / bundle id overrides instead of editing the tracked configs.

## Architecture

### Targets

- **ALVRClient** — the visionOS app. Swift + Metal + RealityKit.
- **ALVREyeBroadcast** — a separate ReplayKit Broadcast Upload Extension (`ALVREyeBroadcast/SampleHandler.swift`) used to mirror the eye/face capture out of the app sandbox.
- **ALVRClientCore.xcframework** — pre-built Rust streaming core (see above). The Swift code calls into it via free C functions (`alvr_initialize`, `alvr_resume`, `alvr_send_*`, `alvr_poll_event`, `alvr_get_foveation_center`, etc.).

### Three immersive spaces in `ALVRClientApp.swift`

`ALVRClientApp` defines a `WindowGroup` (the "Entry" lobby UI) plus three `ImmersiveSpace`s, all sharing a `ContentStageConfiguration` that enables CompositorServices foveation when supported:

1. **`DummyImmersiveSpace`** — short-lived. Runs `DummyMetalRenderer` purely to query FOV/view transforms from CompositorServices, then exits. The result feeds the real renderer.
2. **`RealityKitClient`** — `RealityKitClientView` + `RealityKitClientSystem`. Mixed/progressive immersion. Required path for the passthrough-blended modes.
3. **`MetalClient`** — `MetalClientSystem` driving the `Renderer`. Mixed/full immersion. The lower-level Metal path; also the fallback when RealityKit features aren't needed.

The user picks which space via the Entry UI; only one is live at a time. `clientImmersionStyle` / `realityKitImmersionStyle` are mutated in response to `GlobalSettings.enableProgressive`.

### State and event flow

- `EventHandler.shared` (`EventHandler.swift`) is a long-running singleton that owns the bridge to the Rust core. `handleAlvrEvents` is the main event pump; `handleMdnsBroadcasts` does Bonjour discovery; an internal watchdog restarts the thread if it dies. UI screens observe it as an `EnvironmentObject`.
- `WorldTracker.shared` (`WorldTracker.swift`) owns ARKit world/hand/face tracking and is initialized once before `EventHandler.start()`.
- `GlobalSettingsStore` (`GlobalSettings.swift`) is the persisted user settings model, surfaced as `ALVRClientApp.gStore` and as `@EnvironmentObject`. `Settings.swift` is the in-app settings UI; `Entry/Entry.swift` + `EntryControls.swift` are the lobby/launch UI.
- `ViewModel` (`Model/ViewModel.swift`) holds transient app-scoped state (e.g. `isShowingClient`) shared via SwiftUI `@Environment`.

### Rendering pipeline notes

- Metal shaders live in `ALVRClient/Shaders.metal`; CPU-shared types are in `ShaderTypes.h` (also referenced by the bridging header).
- `VideoHandler.swift` + `NALParser.swift` + `AV1Parser.swift` decode the incoming H.264/HEVC/AV1 stream via VideoToolbox.
- `FFR.swift` implements fixed-foveated reconstruction of the decoded image. `RealityKitEyeTrackingSystem.swift` + per-frame `alvr_get_foveation_center` drive gaze-following foveation (see commit `e1dd8c5`).
- `ChaperoneSystem.swift` draws the playspace boundary; `CameraView.swift` is the passthrough/camera surface.
- USDA assets (`SBSMaterial.usda`, `EyeTrackingMats.usda`) are RealityKit material graphs loaded by the RealityKit path.

### Conventions worth knowing

- Swift: 4-space indent, Swift API Design Guidelines naming.
- Tunables tend to live near the top of each file — search there before adding new constants.
- Renderer/system files keep CPU↔GPU types in sync via `ShaderTypes.h`; if you add a new uniform, update both sides.
- Commit subjects in recent history use short prefixes (`fix:`, `perf:`, `client:`, `WIP`); match that style.
- There is no automated test suite. Validate by building and exercising on visionOS hardware (the streamer side is the PC running [ALVR](https://github.com/alvr-org/ALVR)).
