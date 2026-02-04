---
name: layout-autolayout
description: Auto Layout and Liquid Glass rules for UIKit/AppKit layouts, margins, stack views, and system materials.
metadata:
  short-description: Auto Layout and Liquid Glass rules for UIKit/AppKit UI work.
---

# Layout and Liquid Glass

Use this skill for UI layout or visual styling work on UIKit or AppKit.

## Auto Layout (Required)
All UI layout must use Auto Layout with layout anchors. Frame-based layout and autoresizing masks are not allowed for production UI.

### Requirements
- Use Auto Layout exclusively for layout.
- Prefer UIStackView/NSStackView for linear layouts.
- Prefer layout anchors (NSLayoutAnchor APIs).
- Do not rely on autoresizing masks (translatesAutoresizingMaskIntoConstraints = false).
- Intrinsic size must be computable without overriding intrinsicContentSize. Use subview constraints, hugging/compression, and systemLayoutSizeFitting when needed.

### Layout Margins (Default)
Layout margins are the default constraint reference.

UIKit:
```swift
view.layoutMarginsGuide.leadingAnchor
view.layoutMarginsGuide.trailingAnchor
view.layoutMarginsGuide.topAnchor
view.layoutMarginsGuide.bottomAnchor
```

AppKit:
```swift
view.layoutMarginsGuide.leadingAnchor
view.layoutMarginsGuide.trailingAnchor
view.layoutMarginsGuide.topAnchor
view.layoutMarginsGuide.bottomAnchor
```

### Hard-coded Spacing
- Avoid hard-coded spacing constants (8, 12, 16, etc.).
- Prefer layout margins, intrinsic content size, and system spacing relations.
- Hard-coded constants are allowed only when explicitly required by design.

### Valid Exceptions (Raw Anchors)
- Aligning sibling views
- Building custom controls
- Drawing edge-to-edge backgrounds
- Implementing separators
- Performance-critical measurement-only layout

### Constraint Hygiene
- Keep constraints close to the views they lay out.
- Group and activate constraints together.
- Readability is required.

### Platform Notes
- UIKit: Auto Layout + anchors + margins only.
- AppKit: Auto Layout + anchors + margins only; avoid springs-and-struts.

### UIKit/AppKit Addendum
- Prefer stack views for vertical/horizontal composition; only drop to manual constraints for non-linear or performance-critical layouts.
- Use layoutMarginsGuide and isLayoutMarginsRelativeArrangement with stack views to respect margins by default.
- Use system spacing APIs before hard-coded constants.
- Prefer system controls over custom drawing to preserve native focus/selection behavior.
- Custom views should size via constraints and subview intrinsic sizes, never by overriding intrinsicContentSize.

## Liquid Glass (Required for UI Work)
- Prefer system materials (UIBlurEffect, NSVisualEffectView) where appropriate.
- Respect translucency, depth, and layering.
- Avoid hard, opaque backgrounds unless required for readability.
- Use dynamic system colors; avoid hard-coded colors.
- Motion and transitions should be subtle and platform-appropriate.

### UIKit
- Use system-provided materials and vibrancy where applicable.
- Avoid custom blur implementations.

### AppKit
- Prefer NSVisualEffectView for backgrounds and containers.
- Ensure vibrancy works correctly with text and selection.

### Rule
- UI changes must not break Liquid Glass aesthetics.
- If a design deviates from Liquid Glass, the reason must be explicit.
