---
name: accessibility-keyboard
description: VoiceOver, Dynamic Type, keyboard navigation, focus ring, and accessibility actions for UIKit/AppKit.
metadata:
  short-description: Accessibility and keyboard rules for UIKit + AppKit.
---

# Accessibility and Keyboard

Use this skill when adding UI that must be accessible or keyboard navigable.

## Rules
- Every interactive control must have a clear accessibility label.
- Support Dynamic Type on iOS and system text sizes on macOS.
- Keyboard navigation must work without a pointer device.
- Custom controls must expose accessibility traits and actions.
- Do not reduce contrast or remove focus rings for style.

## UIKit
- Use UIFont.preferredFont(forTextStyle:) and adjustsFontForContentSizeCategory.
- Use UIAccessibility labels, hints, and traits for controls.
- Support external keyboard navigation; keep tab order predictable.
- Use UIAccessibilityCustomAction for custom actions when needed.

UIKit example:

```swift
@MainActor
final class ItemCell: UITableViewCell {

    func configure(title: String) {
        textLabel?.text = title
        textLabel?.font = .preferredFont(forTextStyle: .body)
        textLabel?.adjustsFontForContentSizeCategory = true
        isAccessibilityElement = true
        accessibilityLabel = title
        accessibilityTraits = [.button]
    }
}
```

## AppKit
- Use system fonts and respect user size settings.
- Ensure the key view loop is defined (nextKeyView).
- Keep focus ring visible on interactive controls.
- Use NSAccessibility labels, roles, and actions for custom controls.

AppKit example:

```swift
final class ItemRowView: NSView {

    private let label = NSTextField(labelWithString: "")

    func configure(title: String) {
        label.stringValue = title
        label.font = NSFont.preferredFont(forTextStyle: .body)
        setAccessibilityElement(true)
        setAccessibilityLabel(title)
        setAccessibilityRole(.button)
        focusRingType = .default
    }
}
```
