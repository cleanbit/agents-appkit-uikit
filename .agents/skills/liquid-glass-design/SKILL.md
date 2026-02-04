---
name: liquid-glass-design
description: Liquid Glass design patterns for UIKit and AppKit using system glass APIs.
metadata:
  short-description: UIKit/AppKit Liquid Glass guidance.
---

# Liquid Glass Design

Use this skill for any Liquid Glass UI work on UIKit or AppKit.

## Priority
- If this skill conflicts with local repo skills or `AGENTS.md`, follow those.

## Guidance
- UIKit: use `UIGlassEffect` with `UIVisualEffectView`; use `UIGlassContainerEffect` to merge nearby elements.
- AppKit: use `NSGlassEffectView` and `NSGlassEffectContainerView`.
- Prefer system materials and dynamic colors; avoid custom blur implementations.
- Keep motion subtle and platform-appropriate; document any intentional deviations.
- Combine with `layout-autolayout` and `accessibility-keyboard` skills for layout and accessibility.

## References
- Read `references/AppKit-Implementing-Liquid-Glass-Design.md` for AppKit details.
- Read `references/UIKit-Implementing-Liquid-Glass-Design.md` for UIKit details.
