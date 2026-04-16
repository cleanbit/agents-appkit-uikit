# Liquid Glass Design

Use this reference for any Liquid Glass UI work on UIKit or AppKit.

## Priority
- If this reference conflicts with local repo guidance or `AGENTS.md`, follow those.

## Guidance
- UIKit: use `UIGlassEffect` with `UIVisualEffectView`; use `UIGlassContainerEffect` to merge nearby elements.
- AppKit: use `NSGlassEffectView` and `NSGlassEffectContainerView`.
- Prefer system materials and dynamic colors; avoid custom blur implementations.
- Keep motion subtle and platform-appropriate; document any intentional deviations.
- Combine with `layout-autolayout.md` and `accessibility-keyboard.md` for layout and accessibility.

## References
- Read `references/AppKit-Implementing-Liquid-Glass-Design.md` for AppKit details.
- Read `references/UIKit-Implementing-Liquid-Glass-Design.md` for UIKit details.
