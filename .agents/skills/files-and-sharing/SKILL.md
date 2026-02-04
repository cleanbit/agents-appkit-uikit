---
name: files-and-sharing
description: UIDocumentPicker/NSOpenPanel usage, file coordination, and security-scoped URLs.
metadata:
  short-description: Files and sharing rules for UIKit + AppKit.
---

# Files and Sharing

Use this skill when opening, saving, or importing/exporting files.

## Rules
- Use UIDocumentPicker (iOS) and NSOpenPanel/NSSavePanel (macOS).
- Access security-scoped URLs only within a scoped block.
- Coordinate reads/writes when modifying files.
- File I/O must be off the main thread; UI updates on @MainActor.
- Side controllers perform file parsing and mutation; view controllers only present UI.

## UIKit (UIDocumentPicker)
- Use UIDocumentPickerViewController for user-driven access.
- Start accessing security-scoped resources before reading.

UIKit example:

```swift
@MainActor
final class FilesViewController: UIViewController, UIDocumentPickerDelegate {

    func presentPicker() {
        let picker = UIDocumentPickerViewController(forOpeningContentTypes: [UTType.data])
        picker.delegate = self
        present(picker, animated: true)
    }

    func documentPicker(_ controller: UIDocumentPickerViewController, didPickDocumentsAt urls: [URL]) {
        guard let url = urls.first else { return }
        let accessGranted = url.startAccessingSecurityScopedResource()
        defer {
            if accessGranted { url.stopAccessingSecurityScopedResource() }
        }
        Task {
            _ = try? Data(contentsOf: url)
        }
    }
}
```

## macOS (NSOpenPanel/NSSavePanel)
- Use NSOpenPanel for import and NSSavePanel for export.
- Create and store security-scoped bookmarks when needed.

AppKit example:

```swift
@MainActor
final class FilesController {

    func openFile() {
        let panel = NSOpenPanel()
        panel.allowedContentTypes = [.data]
        panel.begin { response in
            guard response == .OK, let url = panel.url else { return }
            let accessGranted = url.startAccessingSecurityScopedResource()
            defer {
                if accessGranted { url.stopAccessingSecurityScopedResource() }
            }
            Task {
                _ = try? Data(contentsOf: url)
            }
        }
    }
}
```
