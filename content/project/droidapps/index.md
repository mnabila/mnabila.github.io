+++
draft = false
date = '2026-03-14T21:50:16+07:00'
title = 'DroidApps'
type = 'project'
description = 'A technical look at building a Linux shell script that runs Android apps as if they were native desktop applications — using ADB secondary displays, scrcpy mirroring, and rofi for app selection.'
image = ''
repository = 'https://github.com/mnabila/droidapps'
languages = ['bash']
tools = ['scrcpy', 'adb', 'rofi', 'bash']
+++

What if you could launch an Android app on your Linux desktop the same way you launch a native application — pick it from a menu, and it just appears in its own window? No emulator, no Android Studio, no virtual machine. Just your phone connected over USB and a lightweight mirroring tool.

DroidApps is a Bash script that makes this possible. Inspired by [WinApps](https://github.com/Fmstrat/winapps) (which does the same thing for Windows applications via RDP), it uses ADB, scrcpy, and Android's secondary display feature to run individual Android apps in their own desktop windows on Linux.

## Problem Background

As a developer working with both Linux and Android, I frequently needed to interact with Android apps during development and testing. The standard options each had significant drawbacks:

- **Android emulators** are resource-heavy — they consume gigabytes of RAM, require hardware virtualization, and take time to boot. For simply running an app to check something, this is overkill
- **Full-screen scrcpy mirroring** shows the entire phone screen, which means navigating through launchers, switching between apps, and dealing with a phone-shaped window on a desktop-oriented workflow
- **Android-x86 in a VM** solves some problems but introduces others: compatibility issues with ARM-only apps, separate storage, and the overhead of managing a virtual machine

What I wanted was simple: select an Android app from a menu, have it launch on my phone, and see only that app's window on my Linux desktop — as if it were a native application. No emulator boot time, no full-screen phone mirror, no context switching.

## Solution Overview

DroidApps achieves this by combining three existing tools in a way they were not originally designed to work together:

1. **ADB** manages the device connection and launches apps on a specific Android display
2. **Android secondary displays** provide an isolated display surface where an app runs independently from the phone's main screen
3. **scrcpy** mirrors only that secondary display to a desktop window

The result is a single Bash script that orchestrates the entire flow: detect devices, present an app picker, configure the secondary display, launch the selected app on it, and mirror the result to your desktop.

**Tech stack:** Bash, ADB, scrcpy, rofi, awk

**My role:** Sole developer — concept, implementation, and AUR packaging

## System Architecture

The entire project is a single Bash script (~90 lines) that orchestrates four system tools. The architecture is best understood as a pipeline:

```
Device Detection → App Selection → Display Setup → App Launch → Screen Mirroring
     (ADB)           (rofi)          (ADB)          (ADB)        (scrcpy)
```

Each stage handles one concern:

**1. Device Detection (`getDevices`)** — queries ADB for connected devices. If multiple devices are found, it presents a rofi menu for selection. If only one device is connected, it is used automatically.

**2. App Selection (`getPackage`)** — retrieves the list of installed packages from the device. If `aapt` is available on the device, it resolves human-readable app names (e.g., "WhatsApp" instead of `com.whatsapp`). The list is presented through rofi for selection. Once selected, it resolves the app's main activity via `dumpsys package`.

**3. Secondary Display Configuration (`enableSecondaryDisplay`)** — checks whether a simulated secondary display is already active on the device. If not, it creates one at 1920x1080 resolution using Android's `overlay_display_devices` setting. A cleanup trap ensures this display is removed when the script exits.

**4. App Launch and Mirroring** — launches the selected activity on the secondary display using `am start --display`, then starts scrcpy pointed at that specific display ID. The app appears in its own desktop window.

### Cleanup Handling

The script registers a `trap` on `EXIT` to restore the device's display settings when the script terminates — whether by normal exit, Ctrl+C, or error. This prevents orphaned secondary displays from persisting on the device:

```bash
trap cleanup EXIT

function cleanup() {
    if [ "$SECONDARY_DISPLAY_ENABLED" = true ] && [ -n "$DEVICE" ]; then
        adb -s "$DEVICE" shell settings put global overlay_display_devices ""
    fi
}
```

## Key Features

- **No emulator required** — runs apps directly on a physical Android device connected via USB or wireless ADB, with zero emulation overhead
- **App-level isolation** — each app runs on its own secondary display, separate from the phone's main screen. The phone remains fully usable while the app is mirrored to the desktop
- **Human-readable app names** — when `aapt` is available on the device, the app picker shows names like "Telegram" instead of raw package identifiers like `org.telegram.messenger`
- **Multiple device support** — when more than one Android device is connected, the script presents a device selection menu before proceeding
- **Automatic display management** — secondary displays are created on demand and cleaned up automatically on script exit via trap handlers
- **rofi integration** — uses rofi as the selection interface, providing fuzzy search across the app list. This makes finding apps in a list of hundreds fast and intuitive
- **Single-file distribution** — the entire tool is one Bash script with no build step, installable via a single `curl` command or through the AUR on Arch Linux

## Technical Challenges and Solutions

**Resolving app names from package identifiers.** Android's `pm list packages` only returns package names like `com.whatsapp`, which is not user-friendly for an app picker. The solution was to use `aapt dump badging` on each APK to extract the `application-label` field. However, `aapt` is not installed on all devices by default, so the script falls back to raw package names when it is unavailable. The label extraction runs entirely on-device to avoid transferring APK files:

```bash
label=$(aapt dump badging "$apk" 2>/dev/null | grep "application-label:" | head -1 | sed "s/application-label://;s/'//g")
```

**Identifying the correct main activity.** Selecting a package is not enough to launch an app — you need its main activity class. The script extracts this by parsing `dumpsys package` output for the `android.intent.action.MAIN` intent filter. This handles the common case correctly, though apps with non-standard launcher configurations may require manual activity specification.

**Secondary display lifecycle management.** Android's `overlay_display_devices` setting persists across app restarts but not across reboots. The challenge was ensuring the secondary display gets cleaned up when the script exits, regardless of how it exits. The `trap EXIT` pattern handles normal exit, SIGINT (Ctrl+C), and SIGTERM cleanly. The script also checks for an existing secondary display before creating a new one, preventing duplicate displays from stacking up.

**Display ID selection.** Android can have multiple display IDs, and the correct one for the secondary display is not always predictable. Rather than guessing, the script queries `dumpsys display` for all available display IDs and presents them via rofi. This adds one extra selection step but avoids the fragility of hardcoding display IDs.

**Limited app slots per display.** Android's secondary display support has an inherent limitation: each display ID can only host a limited number of concurrent apps. This is a platform constraint, not a script limitation, but it means running many apps simultaneously requires multiple secondary displays. The current implementation does not manage multiple displays automatically — this is a known limitation.

## Lessons Learned

**Composing existing tools beats building from scratch.** DroidApps does not implement device communication, screen mirroring, or UI selection. It orchestrates ADB, scrcpy, and rofi — tools that already do their jobs well. The script's value is in the composition, not the components. This approach kept the implementation under 100 lines and made it immediately reliable.

**Android's display system is more flexible than most people realize.** The `overlay_display_devices` setting and `am start --display` flag are not widely known outside the Android framework team, but they enable powerful use cases when combined with external mirroring tools. Exploring underdocumented platform features often reveals capabilities that solve real problems.

**Shell scripts are underrated for tool integration.** For a project whose entire purpose is connecting four command-line tools in a specific sequence, Bash is the right choice. No build step, no dependencies beyond the tools themselves, and any Linux user can read and modify the script. The temptation to rewrite in a "real" language should be resisted when the shell version is already correct and maintainable.

**Graceful cleanup is non-negotiable for device-modifying scripts.** Any script that changes device state (in this case, creating a secondary display) must clean up after itself. The `trap EXIT` pattern is essential — without it, a crashed or interrupted script would leave the device in a modified state that the user might not know how to revert.

## Conclusion

DroidApps turns a connected Android phone into an app-by-app extension of the Linux desktop. No emulator, no VM, no heavyweight tooling — just ADB, scrcpy, and a Bash script that ties them together.

Installation on Arch Linux:

```bash
paru -S droidapps
```

On other distributions:

```bash
curl -o ~/.local/bin/droidapps https://github.com/mnabila/droidapps/raw/main/droidapps && chmod +x ~/.local/bin/droidapps
```

Prerequisites: `scrcpy`, `adb`, `rofi`, and an Android device connected via USB or wireless ADB. The full source is available on [GitHub](https://github.com/mnabila/droidapps) — it is a single file, so reading the entire implementation takes about five minutes.
