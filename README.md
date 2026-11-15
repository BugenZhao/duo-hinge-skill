# Duo Hinge Skill

An agent skill that folds and unfolds the iPhone Duo simulator from the command line, and resizes an app freely in Device Hub's resizable mode. `simctl`, `devicectl` and XCTest have no way to fold the simulator, so normally someone has to drag the hinge slider in Device Hub by hand.

`scripts/duo-hinge` builds a small helper with the simulator SDK and runs it inside the simulator through `xcrun simctl spawn`. The helper posts the same vendor-defined HID event that Device Hub's hinge slider sends, then reads the angle back through CoreMotion. Device Hub does not need to be running, and no Accessibility permission is needed.

## Install

Clone it into your agent's skills directory, for example:

```bash
git clone https://github.com/BugenZhao/duo-hinge-skill.git ~/.claude/skills/duo-hinge
```

The script also works on its own, without an agent.

## Usage

```bash
scripts/duo-hinge open    # 180 degrees, inner display
scripts/duo-hinge close   # 0 degrees, outer display
scripts/duo-hinge half    # 90 degrees
scripts/duo-hinge set 127 # any angle from 0 to 180
scripts/duo-hinge get     # current angle
scripts/duo-hinge status  # angle, displays, and orientation

scripts/duo-hinge resize start 600x800  # frontmost app on a freely sized display
scripts/duo-hinge resize set 1200x700   # resize it; prints the size it actually got
scripts/duo-hinge resize get
scripts/duo-hinge resize stop
```

Resizing wraps `devicectl device appResize` and reports the actual size, since the system keeps the aspect ratio between 4:3 and 2:1 and the long side between 667 and 1280 points. See [SKILL.md](SKILL.md) for the measured limits.

Set `DUO_HINGE_DEVICE` to a UDID when several iPhone Duo simulators are booted.

See [SKILL.md](SKILL.md) for accuracy notes and tips for testing after a pose change.

## Requirements

- macOS with an Xcode that ships the iPhone Duo simulator
- A booted iPhone Duo simulator

## Caveats

The helper relies on private IOKit and CoreMotion interfaces, so a new Xcode or simulator runtime can break it.

## Credits

The mechanism follows [artemnovichkov/hinge](https://github.com/artemnovichkov/hinge).
