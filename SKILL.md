---
name: duo-hinge
description: Fold and unfold the iPhone Duo simulator from the command line, without Device Hub or any human step. Use when testing an app on the inner display, checking how layouts react to folding, or whenever a task needs the Duo simulator in a specific pose.
---

# Duo Hinge

Neither `simctl`, `devicectl`, nor XCTest can fold the iPhone Duo simulator. `scripts/duo-hinge` runs a small helper inside the simulator that posts the same HID event as Device Hub's hinge slider, then confirms the angle through CoreMotion. Device Hub does not need to be running, and no Accessibility permission is needed.

## Usage

Paths are relative to this skill's folder.

```bash
scripts/duo-hinge open    # 180 degrees, inner display
scripts/duo-hinge close   # 0 degrees, outer display
scripts/duo-hinge half    # 90 degrees
scripts/duo-hinge set 127 # any angle from 0 to 180
scripts/duo-hinge get     # current angle
scripts/duo-hinge status  # angle, displays, and orientation
```

The first run compiles the helper with the simulator SDK and caches it under `$TMPDIR/duo-hinge`, keyed by the source and the selected Xcode.

If commands run inside a sandbox, run this one outside it, because `simctl` cannot reach CoreSimulator from inside a sandbox. Set `DUO_HINGE_DEVICE` to a UDID when several iPhone Duo simulators are booted.

`open` and `close` land exactly on 180 and 0 degrees. An angle in between settles up to about 5 degrees away, on a side that depends on the direction of travel, so 127 settled at 125.1 going up and 129.7 going down. `set` returns on the first reading within 6 degrees, which can come before the hinge settles, so run `get` afterwards when the exact angle matters.

## Testing after a pose change

- Take a new screenshot after each pose change before tapping by coordinates, because the point size changes: 466x678 on the outer display and 951x669 on the inner display.
- `xcrun simctl io <udid> screenshot --display=primary` captures the outer display, and `--display=primary-1` captures the inner one. `--display=internal` fails with `Invalid argument`.
- Only an app built with the iOS 27 SDK or later resizes. An older build stays in a small centered area on the inner display, so check `DTSDKName` in the app's `Info.plist` before judging a layout.

## When it breaks

The helper relies on private interfaces, so a new Xcode or runtime can break it. A failed `set` exits non-zero after 3 seconds. If the simulator stops responding to hinge events, shut it down and boot it again. [artemnovichkov/hinge](https://github.com/artemnovichkov/hinge) tracks the same mechanism and is the first place to compare against.
