# AGENTS

This repository builds iOS (UIKit) and macOS (AppKit) apps in Swift using MVC. iOS includes iPadOS.

## Targets and Entry Points
- This project uses separate iOS and macOS targets. Target membership is the gate for platform-specific files.
- Code/iOS/**: iOS target only.
- Code/macOS/**: macOS target only.
- Code/Core/**: Core framework target only.
- Do not use #if os(...) for whole files. Only small localized branches are allowed when sharing a file is unavoidable.
- macOS must keep Resources/macOS/Base.lproj/MainMenu.xib in the macOS target.
- iOS must use UIScene (SceneDelegate). Do not drive iOS lifecycle solely from UIApplicationDelegate.
- AppDelegate files are per-platform.

## Localization and UI Files
- Use Resources/Localizable.xcstrings only. Do not add .strings or .stringsdict.
- Storyboards/XIBs are allowed only for:
  - Resources/iOS/Base.lproj/Main.storyboard
  - Resources/iOS/Base.lproj/LaunchScreen.storyboard
  - Resources/macOS/Base.lproj/MainMenu.xib
- All other UI must be programmatic.

## Architecture Guardrails
- Controllers coordinate UI. Logic lives in side controllers/services.
- Backend/networking code must be UI-independent and must not reference UIKit/AppKit.
- All shareable, non-UI code lives in Code/Core and the Core framework target.
- The Core framework must not depend on UIKit/AppKit and should avoid platform conditionals.
- Do not introduce shared UI abstractions to force UIKit/AppKit into one API.
- SwiftUI is forbidden. No SwiftUI views, modifiers, or hosting controllers.

## Project Structure and Naming
- Feature-based folders (not by technical layer).
- File name matches the primary type; one primary type per file.
- Types use PascalCase. Methods/properties use lowerCamelCase.
- Core Data/SwiftData model names are singular and explicit.
- Do not commit secrets.

## UX and Visual Rules
- Prefer native controls and behaviors. Respect platform conventions.
- Use SF Symbols for all icons.
- Liquid Glass is required for UI work; use system materials and dynamic colors.

## Logging
- Use OSLog/Logger. Avoid print except temporary local debugging.
- Prefer logging in side controllers/services; UI classes should log only UI lifecycle/navigation when needed.
- Use subsystem + category. Do not log secrets or PII. Use privacy annotations for user data.

## Process and Quality
- Match local patterns and keep diffs small.
- UI updates must happen on the main thread (@MainActor).
- Do not regress accessibility.
- All builds/tests must use XcodeBuildMCP. Do not invoke xcodebuild directly.
- Tests are required for any logic change (unit + UI) and must be run.
- No drive-by refactors.
