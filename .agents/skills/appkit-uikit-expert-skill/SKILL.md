---
name: appkit-uikit-expert-skill
description: Use when building, reviewing, or updating native UIKit, iPadOS/UIKit, or AppKit apps in Swift MVC, especially code that must avoid SwiftUI, keep Core UI-independent, use side controllers/services, apply Liquid Glass, preserve accessibility, and verify with XcodeBuildMCP.
---

# AppKit UIKit Expert Skill

## Core Contract

Use native UIKit and AppKit with MVC. Controllers coordinate UI and navigation; side controllers and services own loading, mutation, validation, formatting, grouping, persistence, networking, and background work. Shared non-UI logic belongs in Core and must not import UIKit or AppKit.

Follow `AGENTS.md` first when working in this repo. If a reference here conflicts with `AGENTS.md`, use `AGENTS.md`.

## Always Enforce

- SwiftUI is forbidden: no SwiftUI views, modifiers, previews, hosting controllers, or StoreKit SwiftUI views.
- Keep platform files target-specific: `Code/iOS/**` for UIKit, `Code/macOS/**` for AppKit, `Code/Core/**` for shared logic.
- Avoid whole-file `#if os(...)`; only use small localized platform branches when sharing is unavoidable.
- Build UI programmatically except the allowed storyboard/XIB entry points listed in `AGENTS.md`.
- Use Liquid Glass for UI work with system APIs, system materials, and dynamic colors.
- Use SF Symbols for icons.
- Keep UI updates on the main thread, using `@MainActor` where appropriate.
- Use OSLog/Logger with subsystem and category. Never log secrets, PII, prompts, pasteboard contents, receipts, or captured content.
- Preserve accessibility, keyboard navigation, focus, Dynamic Type, and platform-native behavior.
- Use XcodeBuildMCP only for builds, tests, simulator runs, logs, and UI verification. Do not invoke `xcodebuild` directly.

## Workflow

1. Read the references that match the task. Do not load every reference by default.
2. Check target membership and platform boundaries before editing.
3. Put logic in side controllers/services first, then wire controllers to display state and forward user actions.
4. Use snapshot-driven updates for lists and stable identifiers for selection, drag/drop, undo, and Core Data.
5. Add or update tests for logic changes. For UI changes, include UI verification where the repo has UI test coverage.
6. Verify with XcodeBuildMCP before claiming the work is complete.

## Reference Map

Start with these references for common work:

| Task | Read |
| --- | --- |
| MVC boundaries, side controllers, backend isolation | `references/side-controllers.md` |
| UIKit/AppKit layout and styling | `references/layout-autolayout.md`, `references/liquid-glass-design.md` |
| Accessibility, keyboard, focus, Dynamic Type | `references/accessibility-keyboard.md` |
| Lists, tables, collections | `references/diffable-data-source.md` |
| Core Data-backed lists | `references/core-data-frc.md`, `references/diffable-data-source.md` |
| Search/filtering | `references/lists-search-filter.md` |
| Drag/drop and reordering | `references/lists-drag-drop.md` |
| Forms, validation, text input | `references/forms-validation-focus.md` |
| Menus, toolbars, commands, shortcuts | `references/menus-toolbars-commands.md` |
| Target/action selector naming | `references/ui-actions-selectors.md` |
| Windowing, scenes, navigation, restoration | `references/windowing-scenes-navigation.md` |
| Files, import/export, security-scoped URLs | `references/files-and-sharing.md` |
| Pasteboard and sharing | `references/pasteboard-sharing.md` |
| Undo and redo | `references/undo-redo.md` |
| Background tasks and long-running work | `references/background-work.md` |
| Swift conventions, Foundation formatting | `references/formatting-foundation.md` |
| Swift concurrency and data-race safety | `references/swift-concurrency-updates.md` |
| Builds and tests | `references/testing-xcodebuildmcp.md` |

Read API/update references only when the task needs that framework or feature:

| Framework or Feature | Read |
| --- | --- |
| App Intents, Spotlight, snippets | `references/appintents-updates.md`, `references/appintents-updates-api.md` |
| Assistive Access | `references/assistive-access-ios.md`, `references/assistive-access-ios-implementation.md` |
| AttributedString | `references/foundation-attributedstring-updates.md`, `references/foundation-attributedstring-updates-api.md` |
| Foundation Models on-device LLM | `references/foundation-models-on-device-llm.md`, `references/foundation-models-on-device-llm-api.md` |
| Liquid Glass platform details | `references/liquid-glass-design-uikit.md`, `references/liquid-glass-design-appkit.md` |
| MapKit place descriptors | `references/mapkit-place-descriptors.md`, `references/mapkit-place-descriptors-api.md` |
| StoreKit | `references/storekit-updates.md`, `references/storekit-updates-api.md` |
| InlineArray and Span | `references/swift-inlinearray-span.md`, `references/swift-inlinearray-span-api.md` |
| SwiftData inheritance | `references/swiftdata-class-inheritance.md`, `references/swiftdata-class-inheritance-api.md` |
| Visual Intelligence | `references/visual-intelligence-ios.md`, `references/visual-intelligence-ios-implementation.md` |

## Review Checklist

- Platform boundaries are clean and target membership is the mechanism for platform-specific files.
- Core and backend code remain UI-independent.
- UI code uses native UIKit/AppKit controls, Auto Layout, Liquid Glass, SF Symbols, and dynamic colors.
- Lists use diffable data sources and stable item identifiers.
- Side controllers/services own logic; controllers own UI coordination.
- Accessibility and keyboard behavior are not regressed.
- Logging uses Logger/OSLog and privacy annotations.
- Tests and XcodeBuildMCP verification match the change risk.
