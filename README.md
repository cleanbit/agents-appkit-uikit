# agents-appkit-uikit

A small, instruction-driven repo for building **iOS (UIKit)** and **macOS (AppKit)** apps in Swift using **MVC** with shared logic in a **Core SPM package**. It is intentionally minimal and currently contains only contributor guidance (see `AGENTS.md`).

## What this repo is for

- Demonstrating and enforcing local engineering conventions for UIKit/AppKit apps.
- Keeping UI logic in controllers and business logic in side controllers/services.
- Sharing platform-agnostic logic via Swift Package Manager (SPM).

## Key principles (short version)

- **MVC**: View/ViewController wires UI; side controllers own data and business rules.
- **Diffable data sources**: Snapshot-driven updates; stable identifiers only.
- **Selection preservation**: Preserve by IDs, not index paths/rows.
- **Auto Layout only**: Layout anchors + layout margins; no autoresizing masks.
- **Liquid Glass UI**: Prefer system materials and translucency.
- **SwiftUI is forbidden**: UIKit/AppKit only.
- **Locale-aware formatting**: No `String(format:)` for numbers.
- **Modern Swift**: `async/await`, `Task.sleep(for:)`, `URL.documentsDirectory`, etc.
- **Tests required**: Logic changes must include deterministic unit tests.

## Project structure (expected)

```
Apps/MyProduct-UIKit/<FeatureName>/...
Apps/MyProduct-AppKit/<FeatureName>/...
Packages/Core/Sources/<FeatureName>/...
Packages/Core/Tests/<FeatureName>/Tests/...
```

## Source of truth

The full, authoritative guidance lives in:

- `AGENTS.md`

If something in this README conflicts with `AGENTS.md`, treat `AGENTS.md` as the source of truth.

