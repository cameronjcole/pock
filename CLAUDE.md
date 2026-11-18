# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Pock is a macOS menu-bar app (Swift/AppKit, deploy target macOS 10.15) that replaces the MacBook Pro Touch Bar contents with a bar of user-installable **widgets**. This is a personal fork of the upstream project, which is unmaintained (last upstream activity ~2021); the goal is to keep it working on the owner's Touch Bar MacBook and add features/widgets as needed.

## Build

Dependencies come from CocoaPods (`Podfile`). `Pods/`, `Podfile.lock` and `Pock.xcworkspace/contents.xcworkspacedata` are gitignored, so a fresh clone must run `pod install` before building, and you must always build the **workspace**, not the `.xcodeproj`.

```sh
pod install
xcodebuild -workspace Pock.xcworkspace -scheme Pock -configuration Debug build
```

- `PockKit` is pulled from `git@github.com:pock/pockkit.git` (SSH). If `pod install` fails on auth, switch that URL to HTTPS in the `Podfile`.
- Schemes: `Pock` (main app), `QLPockWidget` (Quick Look extension), `Relaunch` (helper tool).
- **The Pock target has a build phase that runs `killall Pock` and copies the built app over `/Applications/Pock.app`.** Every successful build replaces the installed copy and kills the running instance.
- A CocoaPods "Check Pods Manifest.lock" phase fails the build if Pods are out of sync — rerun `pod install`.
- `Pock/Bundle/Secrets.plist` is referenced as a build resource but gitignored — a fresh clone must create it (an empty `<dict/>` plist is fine) or the build fails with "Build input file cannot be found". It only holds the AppCenter secret, read in non-DEBUG builds; AppCenter itself was retired by Microsoft in 2025.
- The `Podfile` `post_install` hook bumps pods' deployment targets to 10.15; current Xcode rejects the old ones (Zip ships 10.9).
- All targets use ad-hoc signing (`CODE_SIGN_IDENTITY = "-"`, no team). Don't reintroduce upstream's Developer ID / team `788D9WZ9Z3`.
- The PockKit pod resolves to 0.3.1 (master); upstream Pock source was partly written against unreleased PockKit APIs, so if a PockKit call doesn't compile, check `Pods/PockKit` for the actual signature.

## Lint / tests

- SwiftLint runs as a build phase if installed (`brew install swiftlint`); config in `.swiftlint.yml` (line length 220, type body 250, `Pods` excluded). Run manually with `swiftlint` from the repo root. The code uses inline `// swiftlint:disable … / enable …` pairs for intentional violations.
- There are no test targets.

## Architecture

### Startup and control flow
- `AppDelegate` (`@main`) builds the status-bar menu, then calls `AppController.shared.reloadWidgets { … prepareTouchBar() }`. On quit it calls `tearDownTouchBar()` to restore the system Touch Bar.
- `AppController` (singleton, `NSResponder`) is the hub: owns the `PKTouchBarNavigationController` and root `PockTouchBarController`, handles reload/relaunch/toggle, the double-`ctrl` hotkey (toggles visibility), screen-lock handling, the daily update-check timer, Touch Bar customization palette, and opens all windows via `openController(_:)` (only one non-debug window at a time).
- Cross-component signaling uses `NotificationCenter` names declared next to their owners, e.g. `.shouldReloadPock` (Preferences → full reload), `.didLoadWidgets`, `.didFetchLatestVersions`.

### Widgets (the core abstraction)
- Widgets are **external `.pock` bundles** installed in `~/Library/Application Support/Pock/Widgets` (`kWidgetsPath` in `WidgetsLoader.swift`). None are compiled into this repo — the default ones (Dock, Now Playing, ESC, Status, Control Center, Weather) live in separate repos and are downloaded at runtime.
- A widget bundle's `NSPrincipalClass` must conform to `PockKit.PKWidget`. Required Info.plist keys: `CFBundleIdentifier`, `CFBundleName`/`CFBundleDisplayName`, `CFBundleShortVersionString`, `PKWidgetAuthor`; optional `PKWidgetPreferenceClass` for a preferences pane. Parsed by `PKWidgetInfo` (also reused by the QLPockWidget Quick Look extension to preview `.pock` files).
- `WidgetsLoader` scans the folder (skipping names containing `disabled`) into the static `WidgetsLoader.installedWidgets`; bundles that fail to load are still listed with `loaded == false`.
- `PockTouchBarController` (subclass of PockKit's `PKTouchBarMouseController`) maps each loaded widget's bundle identifier to an `NSTouchBarItem.Identifier`, lazily creating and caching `PKWidgetTouchBarItem`s (an `NSCustomTouchBarItem` that instantiates the principal class and wraps its view in `PKWidgetViewController`, forwarding appear/disappear lifecycle). It also forwards screen-edge mouse/drag events to widgets conforming to `PKScreenEdgeMouseDelegate`.
- `WidgetsInstaller` handles install/uninstall/update: zips are extracted with `Zip` into `Widgets/tmp`, validated via `PKWidgetInfo`, then moved in.
- Much of the public widget API (`PKWidget`, `PKTouchBarController`, navigation controller, screen-edge controller) lives in the **PockKit** pod, not this repo — check `Pods/PockKit` when tracing those types.

### Private Touch Bar APIs
Pock depends on undocumented Apple APIs declared in `Pock/Private/ApplePrivate.h` (exposed via `Bridging-Header.h`): `NSTouchBar presentSystemModal…`, `NSFunctionRow`, `DFR*` functions, etc. `TouchBarHelper` wraps them (presentation mode via `com.apple.touchbar.agent` defaults, dimming, restarting `TouchBarAgent`/`TouchBarServer`) and method-swizzles two `NSFunctionRow` methods at startup. These can break across macOS versions — suspect them first when Touch Bar presentation misbehaves.

### Other pieces
- `Preferences` — typed `subscript` over `UserDefaults` keyed by `Preferences.Keys`, with defaults defined in the getter's `switch`; `launchAtLogin` is backed by `LoginServiceKit` instead of defaults. Add new settings by adding a key and a default case.
- `Updater` / `Downloader` — fetch `latestVersions.json` and default widgets from `pock.app` (`stage.pock.app` in DEBUG). These upstream endpoints may no longer exist; failures are logged, not fatal.
- `Relaunch` — tiny CLI bundled into the app; `AppController.relaunch()` runs it with Pock's PID so it can kill and reopen the app.
- `Roger` — logging (`Roger.debug/info/error`); only enabled in DEBUG. Also captures stdout/stderr for the in-app Debug Console window. Global `print` is overridden to be DEBUG-only.
- UI is AppKit with one `.xib` per view controller (same folder/name) and `TinyConstraints` for code layout. User-facing strings use `"key".localized` against `Pock/UI/Languages/*.lproj/Localisations.strings` (en, de, it, pt-PT).
- DEBUG builds add a "Debug…" status-menu submenu and show a `stage` Dock badge.

## Conventions
- Tabs for indentation (some newer code mixes in spaces — match the surrounding file).
- Files start with the Xcode header comment; sections are marked with `/// …` and `// MARK:` comments.
