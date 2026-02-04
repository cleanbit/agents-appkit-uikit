---
name: visual-intelligence-ios
description: Visual Intelligence integration guidance for UIKit iOS apps.
metadata:
  short-description: Visual Intelligence integration.
---

# Visual Intelligence in iOS

Use this skill when integrating Visual Intelligence APIs or related intent queries.

## Priority
- If this skill conflicts with local repo skills or `AGENTS.md`, follow those.

## Guidance
- Keep descriptor/query logic in Core services; UI should only present results.
- Provide robust display representations and fallbacks for missing data.
- Avoid logging captured content or user data; use privacy annotations if logging is required.
- SwiftUI is forbidden; ignore SwiftUI sections in references.

## References
- Read `references/Implementing-Visual-Intelligence-in-iOS.md` for API details and examples.
