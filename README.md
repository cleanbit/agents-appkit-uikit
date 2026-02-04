# agents-appkit-uikit

A minimal, instruction‑driven sample repo for building **UIKit (iOS/iPadOS)** and **AppKit (macOS)** apps in Swift using **MVC**, with shared logic in a Core framework.

This repo is intentionally compact and focused on demonstrating a clean, native approach to multi‑platform apps without SwiftUI.

## What’s inside

- A single Xcode project with separate iOS and macOS targets
- A shared Core framework for platform‑agnostic logic
- UIKit/AppKit entry points and minimal resources
- Contributor rules in `AGENTS.md` with detailed repo skills in `.agents/skills`

## Project layout

```text
Project/
  Code/
    Core/
    iOS/
    macOS/
  Resources/
    iOS/
    macOS/
  Project.xcodeproj/
```

## Goals

- Keep UI logic in controllers and data/business logic in side controllers/services
- Use diffable data sources and snapshot‑driven updates
- Preserve platform‑native behaviors and accessibility
- Keep shared code independent of UIKit/AppKit

## Conventions

Contributor conventions live in `AGENTS.md`, with detailed rules split into repo skills under `.agents/skills`. If anything in this README conflicts with `AGENTS.md`, treat `AGENTS.md` as the source of truth.

## License

MIT (or your preferred license)
