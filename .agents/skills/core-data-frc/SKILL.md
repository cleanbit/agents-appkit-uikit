---
name: core-data-frc
description: Core Data list rules using NSFetchedResultsController with diffable snapshots, Active Record helpers, and NSManagedObjectID identifiers.
metadata:
  short-description: Core Data list rules using NSFetchedResultsController and diffable snapshots.
---

# Core Data + FRC

Use this skill when Core Data backs a list or when NSFetchedResultsController is involved.

## Rules
- If underlying data is Core Data, the side controller must use NSFetchedResultsController for change tracking.
- Keep the controller public API stable (willChange, didChange, sectioned data). Do not expose FRC types to UI.
- FRC is the change source. Diffable is the UI updater.
- Sectioning may be derived manually in the side controller. Do not expose frc.sections directly unless using sectionNameKeyPath-driven sections.
- Prefer rebuilding the snapshot on FRC changes.
- Use NSManagedObjectID as the diffable item identifier.
- Preserve selection by NSManagedObjectID.

## Active Record Pattern (Core Data Required)
- Core Data entities must expose Active Record helpers on the entity (class or extension).
- Every action has two methods: one that accepts NSManagedObjectContext and one that uses the main context.
- Context-free methods delegate to the context-aware method using the main context.
- Main context must come from an NSPersistentContainer (for example, container.viewContext).
- Use typed errors (enum) for CRUD failures and wrap Core Data errors.
- Context-aware methods must use context.perform or context.performAndWait.

Example: main context creation with NSPersistentContainer

```swift
final class CoreDataStack {

    static let shared = CoreDataStack(modelName: "Model")

    let container: NSPersistentContainer

    var mainContext: NSManagedObjectContext { container.viewContext }

    init(modelName: String) {
        container = NSPersistentContainer(name: modelName)
        container.loadPersistentStores { _, error in
            if let error {
                fatalError("Core Data store failed: \(error)")
            }
        }
    }
}
```

Example: entity Active Record helpers with paired methods

```swift
enum ActiveRecordError: Error {
    case fetchFailed(Error)
    case saveFailed(Error)
}

extension User {

    static var mainContext: NSManagedObjectContext {
        CoreDataStack.shared.mainContext
    }

    static func fetch(in context: NSManagedObjectContext) throws -> [User] {
        var results: [User] = []
        var caughtError: Error?
        context.performAndWait {
            let request: NSFetchRequest<User> = User.fetchRequest()
            do {
                results = try context.fetch(request)
            } catch {
                caughtError = error
            }
        }
        if let caughtError {
            throw ActiveRecordError.fetchFailed(caughtError)
        }
        return results
    }

    static func fetch() throws -> [User] {
        try fetch(in: mainContext)
    }

    static func find(objectID: NSManagedObjectID, in context: NSManagedObjectContext) -> User? {
        var result: User?
        context.performAndWait {
            result = context.object(with: objectID) as? User
        }
        return result
    }

    static func find(objectID: NSManagedObjectID) -> User? {
        find(objectID: objectID, in: mainContext)
    }

    static func insert(in context: NSManagedObjectContext) -> User {
        User(context: context)
    }

    static func insert() -> User {
        insert(in: mainContext)
    }

    func save(in context: NSManagedObjectContext) throws {
        var caughtError: Error?
        context.performAndWait {
            do {
                if context.hasChanges {
                    try context.save()
                }
            } catch {
                caughtError = error
            }
        }
        if let caughtError {
            throw ActiveRecordError.saveFailed(caughtError)
        }
    }

    func save() throws {
        try save(in: Self.mainContext)
    }

    func delete(in context: NSManagedObjectContext) {
        context.performAndWait {
            context.delete(self)
        }
    }

    func delete() {
        delete(in: Self.mainContext)
    }
}
```

## FRC Setup (Uses Active Record Main Context)

```swift
let request: NSFetchRequest<User> = User.fetchRequest()
request.sortDescriptors = [NSSortDescriptor(keyPath: \User.name, ascending: true)]

let frc = NSFetchedResultsController(
    fetchRequest: request,
    managedObjectContext: User.mainContext,
    sectionNameKeyPath: nil,
    cacheName: nil
)
```

Use entity fetch requests for FRC wiring. Active Record fetch() is for direct loads,
not for building an NSFetchedResultsController.

## UsersController example (Core Data via FRC)

```swift
@MainActor
final class UsersController: NSObject, NSFetchedResultsControllerDelegate {

    typealias ResultType = User

    struct Section: Hashable {
        let id: UUID
        let users: [User]
    }

    private let frc: NSFetchedResultsController<User>

    private(set) var sections: [Section] = []

    var fetchedObjects: [ResultType]? {
        frc.fetchedObjects
    }

    var willChange: (() -> Void)?
    var didChange: (() -> Void)?

    init(frc: NSFetchedResultsController<User>) {
        self.frc = frc
        super.init()
        self.frc.delegate = self
    }

    func load() throws {
        try frc.performFetch()
        rebuildSections()
    }

    func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        willChange?()
    }

    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        rebuildSections()
        didChange?()
    }

    func object(at indexPath: IndexPath) -> ResultType {
        sections[indexPath.section].users[indexPath.item]
    }

    func indexPath(forObject object: ResultType) -> IndexPath? {
        for (sectionIndex, section) in sections.enumerated() {
            if let itemIndex = section.users.firstIndex(where: { $0.objectID == object.objectID }) {
                return IndexPath(item: itemIndex, section: sectionIndex)
            }
        }
        return nil
    }

    private func rebuildSections() {
        let users = frc.fetchedObjects ?? []
        let grouped = Dictionary(grouping: users, by: sectionID(for:))
        let orderedIDs: [SectionID] = [.online, .offline]
        sections = orderedIDs.compactMap { id in
            guard let users = grouped[id] else { return nil }
            return Section(id: id, users: users)
        }
    }

    private func sectionID(for user: User) -> SectionID {
        // Replace with your domain grouping logic.
        user.isOnline ? .online : .offline
    }
}
```

## Notes on Sectioning with FRC
- FRC remains the change source for Core Data lists; diffable snapshots still drive the UI.
- If you use sectionNameKeyPath, derive section identifiers from frc.sections?.map { $0.name } and append items in that order.
- If you need custom sectioning, derive sections inside the side controller and expose a controller-owned sections model instead of frc.sections.
