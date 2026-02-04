---
name: side-controllers
description: Rules and patterns for MVC side controllers, change notifications, sectioning, and pull-based UI updates in UIKit/AppKit apps.
metadata:
  short-description: Side controller responsibilities, change notifications, and sectioning.
---

# Side Controllers

Use this skill when adding or modifying side controllers, data loading, or change notifications.

## Responsibilities
- View/ViewController: UI setup, event wiring, snapshot application, navigation.
- Side controllers/services: loading, mutation, grouping/sectioning, business rules.
- No architecture crusades. No mandatory view models.

## Backend Isolated From UI
- Backend/networking code must be usable without any UI layer.
- Backend code must not reference UIKit/AppKit or view controllers.

## Change Flow Rules
- Side controllers may notify changes from any thread.
- UI must pull state and apply updates on the main thread.
- Coalesce or dedupe change notifications. Avoid per-item UI pushes.
- UI updates are pull-based: on change notifications, view controllers query the side controller and rebuild snapshots.
- UI must not receive precomputed view models from the model layer.

## Side Controller Contract
- willChange: called before mutating data.
- didChange: called after data is updated.
- Expose a sectioned, ordered property for the UI.

## UsersController example (non-Core Data)

```swift
@MainActor
final class UsersController {

    struct User: Hashable {
        let id: UUID
        let name: String
        let isOnline: Bool
    }

    typealias ResultType = User

    enum SectionID: Hashable {
        case online
        case offline

        var name: String {
            switch self {
            case .online:
                return "online"
            case .offline:
                return "offline"
            }
        }
    }

    final class SectionInfo: NSObject, NSFetchedResultsSectionInfo {
        let name: String
        let indexTitle: String?
        private let items: [ResultType]

        var numberOfObjects: Int { items.count }
        var objects: [Any]? { items }

        init(name: String, indexTitle: String? = nil, items: [ResultType]) {
            self.name = name
            self.indexTitle = indexTitle
            self.items = items
        }
    }

    struct Section {
        let id: SectionID
        let users: [User]
    }

    var sections: [any NSFetchedResultsSectionInfo]? {
        users.map { section in
            SectionInfo(
                name: section.id.name,
                items: section.users
            )
        }
    }

    var fetchedObjects: [ResultType]? {
        users.flatMap(\.users)
    }

    private(set) var users: [Section] = []   // property may include sections

    var willChange: (() -> Void)?
    var didChange: (() -> Void)?

    func object(at indexPath: IndexPath) -> ResultType {
        users[indexPath.section].users[indexPath.item]
    }

    func indexPath(forObject object: ResultType) -> IndexPath? {
        for (sectionIndex, section) in users.enumerated() {
            if let itemIndex = section.users.firstIndex(of: object) {
                return IndexPath(item: itemIndex, section: sectionIndex)
            }
        }
        return nil
    }

    func setUsers(_ sections: [Section]) {
        willChange?()
        users = sections
        didChange?()
    }

    func load() async {
        // network/file/db/etc
        let allUsers = [
            User(id: UUID(), name: "A", isOnline: true),
            User(id: UUID(), name: "B", isOnline: false),
        ]
        let online = allUsers.filter(\.isOnline)
        let offline = allUsers.filter { !$0.isOnline }
        setUsers([
            Section(id: .online, users: online),
            Section(id: .offline, users: offline),
        ])
    }
}
```

Rule: grouping/sectioning belongs in the side controller (or its helpers), not in the view controller.
