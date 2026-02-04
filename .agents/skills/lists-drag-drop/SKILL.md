---
name: lists-drag-drop
description: Drag and drop rules for UIKit/AppKit lists, including reordering, validation, and stable identifiers.
metadata:
  short-description: Drag/drop rules for UIKit + AppKit lists.
---

# Lists Drag and Drop

Use this skill when adding drag and drop or reordering to list and collection UIs.

## Rules
- Use stable identifiers for all items (UUID, database IDs, NSManagedObjectID).
- Never use index paths as identity.
- Reordering updates must be snapshot-driven and centralized (reloadSnapshot()).
- Validate drops by type before accepting. Reject unknown types.
- Support multi-item drags when the UI allows multi-selection.
- Do not mutate UI state directly from drop callbacks; delegate to side controllers.
- Restore selection by stable IDs after reordering.

## UIKit (UITableView/UICollectionView)
- Use UITableViewDragDelegate/UITableViewDropDelegate or UICollectionViewDragDelegate/DropDelegate.
- Distinguish local reorders from external drops (session.localDragSession).
- Use NSItemProvider with explicit UTType identifiers.

UIKit example (table view):

```swift
@MainActor
final class ItemsViewController: UIViewController, UITableViewDragDelegate, UITableViewDropDelegate {

    private let controller: ItemsController
    private let tableView = UITableView()

    init(controller: ItemsController) {
        self.controller = controller
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    func tableView(_ tableView: UITableView, itemsForBeginning session: UIDragSession, at indexPath: IndexPath) -> [UIDragItem] {
        let item = controller.object(at: indexPath)
        let provider = NSItemProvider(object: item.id.uuidString as NSString)
        let dragItem = UIDragItem(itemProvider: provider)
        dragItem.localObject = item.id
        return [dragItem]
    }

    func tableView(_ tableView: UITableView, canHandle session: UIDropSession) -> Bool {
        session.hasItemsConforming(toTypeIdentifiers: [UTType.text.identifier])
    }

    func tableView(_ tableView: UITableView, dropSessionDidUpdate session: UIDropSession, withDestinationIndexPath indexPath: IndexPath?) -> UITableViewDropProposal {
        if session.localDragSession != nil {
            return UITableViewDropProposal(operation: .move, intent: .insertAtDestinationIndexPath)
        }
        return UITableViewDropProposal(operation: .copy, intent: .insertAtDestinationIndexPath)
    }

    func tableView(_ tableView: UITableView, performDropWith coordinator: UITableViewDropCoordinator) {
        let destination = coordinator.destinationIndexPath
        controller.handleDrop(coordinator, destinationIndexPath: destination)
    }
}
```

## AppKit (NSTableView/NSCollectionView)
- Register explicit pasteboard types.
- Use NSItemProvider for modern drag sources.
- Validate drop types before accepting.

AppKit example (table view):

```swift
@MainActor
final class ItemsViewController: NSViewController {

    private let controller: ItemsController
    private let tableView = NSTableView()

    init(controller: ItemsController) {
        self.controller = controller
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        tableView.registerForDraggedTypes([.string])
    }
}

extension ItemsViewController: NSTableViewDataSource {

    func tableView(_ tableView: NSTableView, pasteboardWriterForRow row: Int) -> NSPasteboardWriting? {
        let item = controller.object(at: IndexPath(item: row, section: 0))
        return item.id.uuidString as NSString
    }

    func tableView(_ tableView: NSTableView, validateDrop info: NSDraggingInfo, proposedRow row: Int, proposedDropOperation dropOperation: NSTableView.DropOperation) -> NSDragOperation {
        guard info.draggingPasteboard.types?.contains(.string) == true else { return [] }
        return .move
    }

    func tableView(_ tableView: NSTableView, acceptDrop info: NSDraggingInfo, row: Int, dropOperation: NSTableView.DropOperation) -> Bool {
        controller.handleDrop(info, row: row)
        return true
    }
}
```
