---
name: assistive-access-ios
description: Assistive Access support guidance for iOS UIKit apps.
metadata:
  short-description: Assistive Access for UIKit.
---

# Assistive Access in iOS

Use this skill when implementing Assistive Access behaviors or UI adaptations.

## Priority
- If this skill conflicts with local repo skills or `AGENTS.md`, follow those.

## Guidance
- Follow system guidelines for simplified layouts, large touch targets, and clear navigation.
- Ensure accessibility labeling and focus behavior remain correct (see `accessibility-keyboard`).
- Keep any Assistive Access detection or state in side controllers; update UI on the main thread.
- SwiftUI is forbidden; ignore SwiftUI sections in references.

## References
- Read `references/Implementing-Assistive-Access-in-iOS.md` for configuration keys and design guidance.
