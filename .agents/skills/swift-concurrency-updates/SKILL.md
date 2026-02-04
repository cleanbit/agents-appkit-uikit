---
name: swift-concurrency-updates
description: Swift 6.2 concurrency updates and data-race safety guidance.
metadata:
  short-description: Swift 6.2 concurrency updates.
---

# Swift Concurrency Updates

Use this skill when reviewing or updating concurrency behavior.

## Priority
- If this skill conflicts with local repo skills or `AGENTS.md`, follow those.

## Guidance
- Enforce Sendable and actor isolation; avoid shared mutable global state.
- Keep UI work on `@MainActor`.
- Prefer structured concurrency and `Task.sleep(for:)` (see `formatting-foundation`).

## References
- Read `references/Swift-Concurrency-Updates.md` for the latest Swift 6.2 changes.
