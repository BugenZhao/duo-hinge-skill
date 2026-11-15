---
name: duo-hinge
description: Fold and unfold the iPhone Duo simulator, and resize an app freely in Device Hub's resizable mode, from the command line without Device Hub or any human step. Use when testing an app on the inner display, checking how layouts react to folding or to arbitrary window sizes, or whenever a task needs the Duo simulator in a specific pose or size.
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

## Resizable mode

Device Hub's resizable mode moves the frontmost app onto a freely sized display, like iPhone Mirroring on macOS 27. `devicectl device appResize` drives it with public commands, and `scripts/duo-hinge resize` wraps them, measuring the size the app really gets.

```bash
scripts/duo-hinge resize start 600x800  # move the frontmost app to a 600x800 pt display
scripts/duo-hinge resize set 1200x700   # resize it while it runs
scripts/duo-hinge resize get            # requested and actual size
scripts/duo-hinge resize stop           # end the session; the app returns to its display
```

Sizes are in points. `start` launches `devicectl device appResize start` in the background, because that command runs until interrupted and the session ends when it exits; `stop` interrupts it. Bring the app to the front before `start`. Without a size, `start` reuses the last requested size. `set` prints the requested and the actual size and adds a note when they differ.

`devicectl device info appResize` reports only the requested size, and the system adjusts requests outside its limits, so the script measures the actual size from `xcrun simctl io <udid> screenshot --display=resizable` (pixels divided by the display's UI scale, 3), waiting until two readings a second apart agree. Use that capture for screenshots of the resized app; `recordVideo --display=resizable` records it.

### Size limits

Measured on the iPhone Duo / iOS 27.2 simulator; the system's Settings app matched Pod Tracker, so these are system limits, not an app's.

- The aspect ratio stays between 4:3 and 2:1, in either orientation. A squarer request becomes 4:3 (900x900 gives 900x675, 1280x1000 gives 1280x960); a longer one gets a short side of half the long side (1280x400 gives 1280x640, 375x1280 gives 640x1280).
- The long side is at least 667 and at most 1280; the short side is at least 375 (450x600 gives 450x667, 1280x1280 gives 1280x960). `devicectl` itself reports 375x375 to 1280x1280.
- Squares are ambiguous: 900x900 became 900x675 in one app and 675x900 in another, and 700x700 gave 525x700. Small squares (375x375 to 500x500) all become 667x375.
- Requests inside these limits apply exactly, for example 800x600, 600x800, 1280x720, 720x1280, 1100x800 and 375x700.
- After starting in portrait, the first request for a landscape size lands on a 3:4 portrait size (600x800, then 1200x700, gives 525x700); the same request again applies. `resize set` retries once when the actual size differs, so it only reports genuine adjustments.

Pick sizes inside the limits when a test needs an exact size, and read the actual size before tapping by coordinates.

The resizable display also exists on other iPhone simulators (an iPhone 18 Pro has one; an iPad Pro does not), so `DUO_HINGE_DEVICE` can point the resize commands at another booted iPhone. Those were not characterized: one iPhone 18 Pro session showed 450x667 for a 900x600 request.

## When it breaks

The hinge helper relies on private interfaces, so a new Xcode or runtime can break it; the resize commands use only `devicectl` and `simctl`. A failed `set` exits non-zero after 3 seconds. If the simulator stops responding to hinge events, shut it down and boot it again. [artemnovichkov/hinge](https://github.com/artemnovichkov/hinge) tracks the same mechanism and is the first place to compare against.
