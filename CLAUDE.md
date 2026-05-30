# CLAUDE.md

Guidance for Claude Code working in the visionOS client. For build commands, coding
style, and the PR/commit guide, see **`AGENTS.md`** — this file covers architecture,
signing, and the cross-repo pairing rules that aren't in there. The umbrella
`../CLAUDE.md` covers the relationship with the `ALVR/` Rust repo.

## What this is

The visionOS Swift / Metal / RealityKit client (`ALVRClient.xcodeproj`). It does not
contain the streaming logic itself — that lives in the Rust `alvr_client_core` crate
(in the sibling `ALVR/` repo), compiled and wrapped as
`ALVRClient/ALVRClientCore.xcframework` and called through
`ALVRClient/ALVRClient-Bridging-Header.h`. When the Rust C ABI changes, re-run
`build_and_repack.sh` to rebuild the xcframework (see `../CLAUDE.md`).

This is a **fork**: `origin` = `scottso/alvr-visionos`, `upstream` =
`alvr-org/alvr-visionos`.

## Local signing — Override.xcconfig

Personal signing identity is kept **out of the repo**. `ALVRClient.xcconfig` defines
placeholder variables and the project/entitlements reference them indirectly:

- `ALVR_DEVELOPMENT_TEAM` (empty default) → `DEVELOPMENT_TEAM`
- `ALVR_BUNDLE_ID` (`alvr.client`) → `PRODUCT_BUNDLE_IDENTIFIER` in `project.pbxproj`
  (`$(ALVR_BUNDLE_ID)`, and `$(ALVR_BUNDLE_ID).ALVREyeBroadcast` for the extension)
- `ALVR_APP_GROUP` (`group.alvr.client.ALVR`) → the App Group in both
  `ALVRClient.entitlements` and `ALVREyeBroadcast.entitlements` (`$(ALVR_APP_GROUP)`)

Override these locally in **`Override.xcconfig`** (gitignored, `#include?`-d last in
`ALVRClient.xcconfig`, so it wins). Put your real team / bundle ID / app group there —
**never** commit personal values into `ALVRClient.xcconfig` or `project.pbxproj`. The
repo defaults keep it buildable without an Override file. Verify resolution with
`xcodebuild -showBuildSettings`.

Notes:
- `com.apple.developer.low-latency-streaming` (in `ALVRClient.entitlements`) requires a
  **paid** Apple developer team; personal/free teams can't sign it.
- A plain compile check works without signing:
  `xcodebuild ... CODE_SIGNING_ALLOWED=NO build`. A signed/installable build additionally
  needs a Vision Pro **registered to the team** (connect/pair the device first).

## PSVR2 Sense controllers

PSVR2 Sense controller support (6DoF pose via `AccessoryTrackingProvider`, plus the
button/stick/trigger/grip mapping) lives in `ALVRClient/WorldTracker.swift`. The code
path is gated behind the **`XCODE_BETA_26`** compilation condition (set in the Debug
config's `SWIFT_ACTIVE_COMPILATION_CONDITIONS`); a build without it omits accessory
tracking and profile registration. Requires the Xcode 26 SDK / visionOS 26.

For controllers to actually input, the client registers the interaction profile with a
populated `input_ids` list (the per-hand path-id arrays) — these must stay in lockstep
with the `alvr_send_button` calls in the PSVR2 branch.

To pair and test: set controller emulation to **PSVR2Sense** in the streamer dashboard.
The streamer must be **built from the sibling `scottso/ALVR` fork** — both for the
`PSVR2Sense` server support (upstream PR #2871, `master`-only, not in stable v20.14.1)
and to match the wire protocol baked into this client's xcframework (including the
fork's eye-tracked foveation changes). See `../CLAUDE.md`.
