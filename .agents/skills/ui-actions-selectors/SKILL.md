---
name: ui-actions-selectors
description: Enforces a single target/action handler convention for UI buttons across UIKit and AppKit.
metadata:
  short-description: Always use didSelectButton(_ sender: Any?) for target/action handlers.
---

# UI Actions and Selectors

Use this skill when wiring any target/action selector in UIKit or AppKit.

## Required Convention (All Target/Action Selectors)
For any target/action selector (buttons, menu items, segmented controls, toolbar items, etc.),
always use this handler name and signature:

```swift
@objc private func didSelectButton(_ sender: Any?) {
    // handle action
}
```

No other naming is allowed for target/action selectors.

## Gesture Recognizer Exception
NSGestureRecognizer handlers are allowed to use explicit handle* names:

Generic:
```swift
@objc private func handleGestureRecognizer(_ gestureRecognizer: NSGestureRecognizer) {
    // handle gesture
}
```

NSClickGestureRecognizer:
```swift
@objc private func handleClickGestureRecognizer(_ gestureRecognizer: NSClickGestureRecognizer) {
    // handle click gesture
}
```

## Wiring Examples

AppKit:
```swift
button.target = self
button.action = #selector(didSelectButton(_:))
```

UIKit:
```swift
button.addTarget(self, action: #selector(didSelectButton(_:)), for: .touchUpInside)
```

## Disallowed Examples
- savePressed()
- buttonPressed()
- saveButtonTapped(_:)
- saveButtonClicked(_:)
- handleSave(_:)
