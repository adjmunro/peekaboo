# LaserGoose

Reusable GUI feedback loop for Claude Code. Captures any running app's window, analyses it, and iterates — without manual screenshots or third-party tools.

Works across:
- **macOS native** apps (Swift, AppKit, SwiftUI)
- **Compose Desktop** JVM apps (appear as macOS windows)
- **iOS Simulator**
- **Android Emulator / device**

## Installation

```zsh
# Make scripts executable
chmod +x ~/Developer/lasergoose/lasergoose
chmod +x ~/Developer/lasergoose/platforms/*.sh
chmod +x ~/Developer/lasergoose/builds/*.sh

# Optional: add to PATH
echo 'export PATH="$HOME/Developer/lasergoose:$PATH"' >> ~/.zshrc
```

## Usage

```
lasergoose [OPTIONS]

Options:
  --app      <name>                App/process display name  (default: $LASERGOOSE_APP or "ThunderSloth")
  --platform <macos|ios-sim|android>  Capture method         (default: $LASERGOOSE_PLATFORM or "macos")
  --build    <xcode|spm|gradle|none>  Build before capture   (default: none)
  --scheme   <name>                Xcode scheme (for xcode build)
  --project  <path>                .xcodeproj/.xcworkspace/project dir path
  --hot                            Hot-reload instead of full build
  --out      <path>                Output PNG path            (default: /tmp/lasergoose.png)
```

## Environment variables

Set these in `.envrc` or a shell alias so you never need to type flags:

| Variable | Default | Description |
|---|---|---|
| `LASERGOOSE_APP` | `ThunderSloth` | App/process display name |
| `LASERGOOSE_PLATFORM` | `macos` | Capture platform |
| `LASERGOOSE_BUILD` | `none` | Build step |
| `LASERGOOSE_SCHEME` | _(none)_ | Xcode scheme |
| `LASERGOOSE_PROJECT` | _(none)_ | Project path |
| `LASERGOOSE_OUT` | `/tmp/lasergoose.png` | Output file |

## Examples

```zsh
# Screenshot only (app already running)
lasergoose --app ThunderSloth --platform macos

# Build then screenshot — Xcode
lasergoose --app ThunderSloth --platform macos \
     --build xcode \
     --scheme ThunderSlothApp \
     --project ~/Developer/ThunderSloth/ThunderSloth/ThunderSloth.xcodeproj

# Build then screenshot — SPM CLI tool
lasergoose --app my-tool --platform macos \
     --build spm \
     --project ~/Developer/my-tool

# iOS Simulator (captures whatever app is on screen)
lasergoose --platform ios-sim

# Android Emulator — build, install, screenshot
lasergoose --app MyAndroidApp --platform android \
     --build gradle \
     --project ~/Developer/my-android-app

# ThunderSloth alias (add to ~/.zshrc, [project]/.env, or justfile)
alias lasergoose-thundersloth='lasergoose --app ThunderSloth --platform macos --build xcode \
  --scheme ThunderSlothApp \
  --project ~/Developer/ThunderSloth/ThunderSloth/ThunderSloth.xcodeproj'
```

## How it works

### macOS capture (`platforms/macos.sh`)

Uses a Swift one-liner (compiled and cached by the system on first run) to query `CGWindowListCopyWindowInfo` for the window ID, then `screencapture -l <id>`:

- Works for **background windows** — reads from the compositor, not the framebuffer.
- Works across Spaces and when windows are partially covered.
- Does **not** reliably capture minimised (Dock) windows.

### iOS Simulator capture (`platforms/ios-sim.sh`)

```zsh
xcrun simctl io booted screenshot /tmp/lasergoose.png
```

### Android capture (`platforms/android.sh`)

```zsh
adb exec-out screencap -p > /tmp/lasergoose.png
```

Uses `~/Library/Android/sdk/platform-tools/adb` (standard Android Studio location).

## Incremental builds vs hot-reload

Incremental builds (`xcodebuild build`, `swift build`, `./gradlew assembleDebug`) are the default. Build systems cache everything and only recompile changed files — typically 2–5 s for small UI changes.

`--hot` is available for trivial cosmetic tweaks but **cannot handle** struct/class layout changes, new model fields, new actors, or anything that changes type definitions.

## Dependencies

### Native to macOS (no install required)
| Tool | Purpose |
|---|---|
| `osascript` | Runs AppleScript to activate apps and send keystrokes |
| `screencapture` | Captures a specific window by ID on macOS |
| `swift` | Compiles and runs the inline Swift window-query snippet; builds SPM projects |
| `xcrun` / `simctl` | Controls the iOS Simulator and takes screenshots |
| `xcodebuild` | Builds Xcode projects |
| `awk`, `base64`, `sleep` | Standard shell utilities for parsing output, encoding images, and timing |
| CoreGraphics, Foundation | macOS frameworks used by the inline Swift snippet to query window info |

### Third-party (must be installed separately)
| Tool | Install | Purpose |
|---|---|---|
| `adb` (Android Debug Bridge) | Android Studio / Android SDK | Communicates with Android devices and emulators, captures screenshots |
| `xcpretty` | `gem install xcpretty` | Formats `xcodebuild` output (optional — build continues without it) |

> **Android SDK path:** `adb` is expected at `~/Library/Android/sdk/platform-tools/adb` (standard Android Studio location).

## Claude Code integration

After each code change, Claude runs:

```zsh
lasergoose-thundersloth
# then reads /tmp/lasergoose.png via the Read tool
```

Claude analyses the image, applies the next diff, and repeats.
