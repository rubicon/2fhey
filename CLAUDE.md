# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

2FHey is a macOS menu-bar app (Swift/AppKit + SwiftUI, Xcode project, no Makefile/CI) that watches incoming iMessage or Google Messages SMS, extracts one-time codes, copies them to the clipboard, and optionally auto-pastes (Cmd+V via Accessibility). Upstream is `SoFriendly/2fhey`; this checkout is a fork (`origin` = `rubicon/2fhey`, `upstream` = `SoFriendly/2fhey`).

## Commands

Targets: `TwoFHey` (the app, product name `2FHey`), `AutoLauncher` (login-item helper), `TwoFHeyTests`. SwiftPM deps (HotKey, SQLite.swift) resolve automatically.

```bash
# Build
xcodebuild build -project TwoFHey.xcodeproj -scheme TwoFHey -destination 'platform=macOS' CODE_SIGNING_ALLOWED=NO MACOSX_DEPLOYMENT_TARGET=12.4

# All tests
xcodebuild test -project TwoFHey.xcodeproj -scheme TwoFHey -destination 'platform=macOS' CODE_SIGNING_ALLOWED=NO MACOSX_DEPLOYMENT_TARGET=12.4

# Single test
xcodebuild test -project TwoFHey.xcodeproj -scheme TwoFHey -destination 'platform=macOS' CODE_SIGNING_ALLOWED=NO MACOSX_DEPLOYMENT_TARGET=12.4 -only-testing:TwoFHeyTests/OTPParserTests/testGoogleFormat
```

`MACOSX_DEPLOYMENT_TARGET=12.4` is needed with current Xcode: the `AutoLauncher` target is pinned to 10.15, which modern Xcode rejects (supported range starts at 12.0). Do not change the pbxproj to fix this unless that is the task. The single-test command above was run and passes.

There is no linter or formatter config. The tests are `@testable import _FHey` (the module name for `2FHey`) and exercise `OTPParser` only.

## Architecture

The pipeline is: **message source -> `OTPParser` -> `AppDelegate` -> clipboard/overlay/paste**.

- **`MessageSource` protocol** (`Models/Message.swift`) is the seam. `AppDelegate.startMessageSource()` picks an implementation from `AppStateManager.shared.messagingPlatform` and subscribes to its `messagesPublisher` (Combine; elements are `MessageWithParsedOTP`). Each source takes the shared `OTPParser` and parses before publishing, so only messages with a code reach `AppDelegate`.
  - `MessageManager` (iMessage): opens `~/Library/Messages/chat.db` via SQLite.swift, watches the `-wal` file with a `DispatchSource` (debounced) rather than polling, tracks a last-processed ROW ID, and decodes `attributedBody` blobs when `text` is empty. Needs Full Disk Access. Can mark messages read.
  - `GoogleMessagesManager` (Android SMS): has no database. It receives notifications from `NotificationServer`, a local TLS WebSocket/HTTP server on `127.0.0.1:2847`. The page-side bridge is `Resources/google-messages-inject.js`, which intercepts web `Notification`s inside the Google Messages wrapper app and forwards them over `wss://127.0.0.1:2847`. `GoogleMessagesSetupService` downloads/installs that wrapper app and generates the TLS certificate. `markMessageAsRead` is a no-op here.
- **`AppDelegate`** (`main.swift` constructs it by hand, there is no `@main`) owns the status-bar item and menu, the global hotkey (Shift+Cmd+E resync, via HotKey), overlay vs native-notification display, and clipboard restore-after-delay. `ParsedOTP.copyToClipboard()` returns the previous clipboard so it can be restored.
- **`AppStateManager`** is the single settings/permissions facade over `UserDefaults` (keys are prefixed). Add new settings there, not ad hoc.
- **`AutoLauncher`** is a separate login-item bundle that launches the main app and is killed via `TwoFHeyKillLauncherNotification`.

### The parser (`OTPParser/OTPParser.swift`)

- Detection is keyword-gated, then pattern-driven. Priority order: `custom-patterns.json` entries (service-specific, can return long numerics), then per-language patterns, then generic extraction.
- **Every candidate must pass `NumberGuard`**, which rejects phone numbers, the sender's own number, money, times, dates, ordinals and decimals. The invariant tested in `TwoFHeyTests` is that a phone number is never returned as a code. `parse(_:sender:)` never searches the sender string for codes. Preserve this when touching extraction.
- A plausible code is 4-8 digits, or 4-10 chars when mixed with letters. URLs are stripped before matching.

### Remotely updated pattern files

Keywords and patterns live in JSON, not Swift: `TwoFHey/OTPKeywords/{en,fr,zh,es,de,pt,he}.json` plus `custom-patterns.json`. At launch the parser loads cache (`~/Library/Caches/OTPKeywords/`) then bundle, then fetches fresh copies from `raw.githubusercontent.com/SoFriendly/2fhey/main/TwoFHey/OTPKeywords/`, validating before caching. The cache is wiped when the app version changes.

Consequences:
- Merging a JSON change to **upstream `main`** ships it to all users on next launch, with no release. Regex mistakes there are live. This fork's `main` is not what the app fetches: the URL is hardcoded to `SoFriendly/2fhey`.
- Adding a language means adding the file under `OTPKeywords/` **and** the filename to `OTPParser.languageFiles`.
- `OTPKeywords` is a synchronized folder group in the Xcode project, so new JSON files are bundled without editing the pbxproj.

## Gotchas

- `README.md` is partly stale: it says `SimpleOTPParser.swift` and `OTPParserContants.swift`/`OTPParserConstants.knownServices`; the parser is now `OTPParser/OTPParser.swift` with no known-services list. `AppConfig.json` (the old 130+ service regex config) is deprecated and unused by the parser.
- Running it needs Full Disk Access (Messages DB) and Accessibility (auto-paste, hotkey) granted to the built binary. A build phase runs `tccutil reset Accessibility` on the bundle id on every build.
- `Google Messages.dmg` at the repo root is a committed binary used by the setup flow.

## Fork / PR workflow

Upstream has no CONTRIBUTING file. Upstream PRs should be code-only: do not include this `CLAUDE.md` in them.
