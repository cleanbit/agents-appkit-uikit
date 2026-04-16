# Foundation AttributedString Updates

Use this reference when working with `AttributedString` or migrating styled text.

## Priority
- If this reference conflicts with local repo guidance or `AGENTS.md`, follow those.

## Guidance
- Prefer `AttributedString` for new styled text; bridge to `NSAttributedString` only at UIKit/AppKit boundaries.
- Keep formatting logic in Core or side controllers; UI should only render results.
- Use locale-aware formatting from `formatting-foundation.md`.
- SwiftUI is forbidden; ignore any SwiftUI guidance in references.

## References
- Read `references/Foundation-AttributedString-Updates.md` for API updates and examples.
