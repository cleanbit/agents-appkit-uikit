---
name: lists-search-filter
description: Search and filtering rules for UIKit/AppKit list UIs with debounced updates and Core Data predicate patterns.
metadata:
  short-description: Search/filter rules for UIKit + AppKit lists.
---

# Lists Search and Filter

Use this skill when adding search or filtering to list and collection UIs on UIKit or AppKit.

## Rules
- Search state and filtering logic live in side controllers.
- View controllers forward query text only; they do not compute filtered sections.
- All list updates are snapshot-driven and centralized in reloadSnapshot().
- Preserve selection and focus by stable IDs, never index paths.
- Debounce filtering with cancellation. Always cancel the previous task.
- Use Task.sleep(for:) with Duration. Never use Task.sleep(nanoseconds:).
- An empty or whitespace-only query means no filter.
- UI updates must happen on the main thread (@MainActor).
- Core framework filtering must not reference UIKit/AppKit types.

## UIKit (UISearchController)
- Use UISearchController in the navigation item.
- Conform to UISearchResultsUpdating and forward searchBar.text to the side controller.
- Debounce in the side controller (recommended) to keep behavior consistent across UIs.

UIKit example:

```swift
@MainActor
final class UsersViewController: UIViewController, UISearchResultsUpdating {

    private let usersController: UsersController
    private let searchController = UISearchController(searchResultsController: nil)

    init(usersController: UsersController) {
        self.usersController = usersController
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        navigationItem.searchController = searchController
        searchController.searchResultsUpdater = self

        usersController.didChange = { [weak self] in
            self?.reloadSnapshot(preserveSelection: true)
        }
    }

    func updateSearchResults(for searchController: UISearchController) {
        usersController.setQuery(searchController.searchBar.text ?? "")
    }
}
```

Side controller debounce example:

```swift
@MainActor
final class UsersController {

    struct User: Hashable {
        let id: UUID
        let name: String
    }

    struct Section {
        let id: UUID
        let users: [User]
    }

    private var allUsers: [User] = []
    private(set) var sections: [Section] = []
    private var pendingSearchTask: Task<Void, Never>?

    var willChange: (() -> Void)?
    var didChange: (() -> Void)?

    func setQuery(_ query: String) {
        let trimmed = query.trimmingCharacters(in: .whitespacesAndNewlines)
        pendingSearchTask?.cancel()
        pendingSearchTask = Task { [weak self] in
            try? await Task.sleep(for: .milliseconds(250))
            await self?.applyQuery(trimmed)
        }
    }

    private func applyQuery(_ query: String) {
        let filtered = query.isEmpty
            ? allUsers
            : allUsers.filter { $0.name.localizedCaseInsensitiveContains(query) }
        setUsers(filtered)
    }

    private func setUsers(_ users: [User]) {
        willChange?()
        sections = [Section(id: UUID(), users: users)]
        didChange?()
    }
}
```

## AppKit (NSSearchToolbarItem + NSSearchField)
- Use NSSearchToolbarItem to provide an NSSearchField.
- Use NSSearchFieldDelegate for per-keystroke updates.
- If you wire target/action, follow the didSelectButton(_:) naming rule.

AppKit example:

```swift
@MainActor
final class UsersViewController: NSViewController, NSSearchFieldDelegate {

    private let usersController: UsersController
    private var searchToolbarItem: NSSearchToolbarItem?

    init(usersController: UsersController) {
        self.usersController = usersController
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    func configureSearchToolbarItem(_ item: NSSearchToolbarItem) {
        searchToolbarItem = item
        let field = item.searchField
        field.placeholderString = "Search"
        field.delegate = self
    }

    func controlTextDidChange(_ obj: Notification) {
        guard let field = obj.object as? NSSearchField else { return }
        usersController.setQuery(field.stringValue)
    }
}
```

## Core Data + FRC Predicate Updates
- When using NSFetchedResultsController, update fetchRequest.predicate and performFetch().
- Wrap predicate updates with willChange/didChange and rebuild sections.
- Use NSManagedObjectID as diffable identifiers.

Example:

```swift
@MainActor
final class UsersController: NSObject, NSFetchedResultsControllerDelegate {

    private let frc: NSFetchedResultsController<User>
    private let logger = Logger(subsystem: "App", category: "UsersController")
    private var pendingSearchTask: Task<Void, Never>?

    var willChange: (() -> Void)?
    var didChange: (() -> Void)?

    func setQuery(_ query: String) {
        let trimmed = query.trimmingCharacters(in: .whitespacesAndNewlines)
        pendingSearchTask?.cancel()
        pendingSearchTask = Task { [weak self] in
            try? await Task.sleep(for: .milliseconds(250))
            await self?.applyQuery(trimmed)
        }
    }

    private func applyQuery(_ query: String) {
        willChange?()
        frc.fetchRequest.predicate = query.isEmpty
            ? nil
            : NSPredicate(format: "name CONTAINS[cd] %@", query)
        do {
            try frc.performFetch()
        } catch {
            logger.error("Fetch failed: \(error.localizedDescription)")
        }
        rebuildSections()
        didChange?()
    }
}
```
