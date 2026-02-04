---
name: appintents-updates
description: App Intents updates and system integration guidance for UIKit/AppKit apps.
metadata:
  short-description: App Intents system integrations.
---

# App Intents Updates

Use this skill when adding or updating App Intents, snippets, Spotlight integration, or related system behaviors.

## Priority
- If this skill conflicts with local repo skills or `AGENTS.md`, follow those.

## Guidance
- Keep intent logic UI-independent; prefer Core services or side controllers for shared logic.
- Ensure intent execution is fast, deterministic, and cancellable.
- Use OSLog/Logger with privacy annotations; never log user content or secrets.
- Prefer platform-native UI (UIKit/AppKit); SwiftUI is forbidden in this repo.

## References
- Read `references/AppIntents-Updates.md` for detailed API changes and examples.
