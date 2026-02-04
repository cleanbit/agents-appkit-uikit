---
name: undo-redo
description: Undo manager usage, grouping, selection restore, and safe mutation hooks for UIKit/AppKit.
metadata:
  short-description: Undo/redo rules for UIKit + AppKit.
---

# Undo and Redo

Use this skill when adding undoable edits or restoring state after undo/redo.

## Rules
- Use UndoManager/NSUndoManager for user-visible undoable changes.
- Group related edits into a single undo action.
- Register undo in side controllers immediately after mutation.
- Restore selection by stable IDs after undo/redo.
- Do not register undo for derived or transient UI state.

## UIKit
- Use the responder chain to expose undo when appropriate.
- Keep undo registration in side controllers; view controllers trigger mutations only.

UIKit example:

```swift
final class ItemsController {

    private let undoManager: UndoManager
    private(set) var items: [Item] = []

    init(undoManager: UndoManager) {
        self.undoManager = undoManager
    }

    func rename(itemID: UUID, to name: String) {
        guard let index = items.firstIndex(where: { $0.id == itemID }) else { return }
        let oldName = items[index].name
        items[index].name = name

        undoManager.registerUndo(withTarget: self) { target in
            target.rename(itemID: itemID, to: oldName)
        }
        undoManager.setActionName("Rename")
    }
}
```

## AppKit
- NSUndoManager is available from the window; register with that manager.
- Group edits with beginUndoGrouping()/endUndoGrouping() when needed.

AppKit example:

```swift
final class ItemsController {

    private let undoManager: UndoManager

    init(undoManager: UndoManager) {
        self.undoManager = undoManager
    }

    func applyBatch(_ changes: [Change]) {
        undoManager.beginUndoGrouping()
        for change in changes {
            apply(change)
        }
        undoManager.endUndoGrouping()
    }

    private func apply(_ change: Change) {
        // mutation + registerUndo
    }
}
```
