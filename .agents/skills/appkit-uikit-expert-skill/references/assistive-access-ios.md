# Assistive Access in iOS

Use this reference when implementing Assistive Access behaviors or UI adaptations.

## Priority
- If this reference conflicts with local repo guidance or `AGENTS.md`, follow those.

## Guidance
- Follow system guidelines for simplified layouts, large touch targets, and clear navigation.
- Ensure accessibility labeling and focus behavior remain correct (see `accessibility-keyboard.md`).
- Keep any Assistive Access detection or state in side controllers; update UI on the main thread.
- SwiftUI is forbidden; ignore SwiftUI sections in references.

## References
- Read `references/Implementing-Assistive-Access-in-iOS.md` for configuration keys and design guidance.
