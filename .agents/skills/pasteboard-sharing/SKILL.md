---
name: pasteboard-sharing
description: UIPasteboard/NSPasteboard patterns, data types, and privacy-safe logging.
metadata:
  short-description: Pasteboard rules for UIKit + AppKit.
---

# Pasteboard and Sharing

Use this skill when copying, pasting, or sharing data via the pasteboard.

## Rules
- Use explicit UTType identifiers for all data.
- Validate pasteboard contents before using them.
- Prefer plain text or structured, versioned types.
- Do not log pasteboard contents. Use privacy annotations if logging metadata.
- Side controllers handle data parsing; view controllers only trigger actions.

## UIKit (UIPasteboard)
- Use UIPasteboard.general for user-facing copy/paste.
- Use NSItemProvider for multi-item sharing.

UIKit example:

```swift
final class ShareController {

    func copyText(_ text: String) {
        UIPasteboard.general.setValue(text, forPasteboardType: UTType.plainText.identifier)
    }

    func pasteText() -> String? {
        UIPasteboard.general.value(forPasteboardType: UTType.plainText.identifier) as? String
    }
}
```

## AppKit (NSPasteboard)
- Use NSPasteboard.general for user-facing copy/paste.
- Declare readable and writable types explicitly.

AppKit example:

```swift
final class ShareController {

    func copyText(_ text: String) {
        let pasteboard = NSPasteboard.general
        pasteboard.clearContents()
        pasteboard.setString(text, forType: .string)
    }

    func pasteText() -> String? {
        NSPasteboard.general.string(forType: .string)
    }
}
```
