---
name: diffable-data-source
description: Guidance for UIKit/AppKit diffable data sources, snapshot rebuilding, stable identifiers, and selection preservation in list/collection UIs.
metadata:
  short-description: Diffable data source rules and snapshot patterns for UIKit/AppKit lists.
---

# Diffable Data Source

Use this skill when building or updating any list or collection UI on UIKit or AppKit.

## Rules
- Use diffable data sources on both platforms.
- iOS: UITableViewDiffableDataSource, UICollectionViewDiffableDataSource
- macOS: NSTableViewDiffableDataSource, NSCollectionViewDiffableDataSource
- Snapshot-driven updates only. Something changes -> rebuild snapshot -> apply snapshot.
- Item identifiers must be stable (UUID, database primary key, NSManagedObjectID).
- Never use array indices as identifiers.
- Preserve selection and focus by IDs, not index paths.

## View Controller Pattern
- Configure the table/collection and diffable data source.
- Capture selection in willChange.
- Rebuild and apply snapshot in didChange.
- Restore selection after apply.
- Keep snapshot logic in one place per screen (reloadSnapshot()).

### UIKit example (collection view; tables are analogous)

```swift
@MainActor
final class UsersViewController: UIViewController {

    private let usersController: UsersController

    private var collectionView: UICollectionView!
    private var dataSource: UICollectionViewDiffableDataSource<UsersController.SectionID, UUID>!

    private var pendingSelectedIDs: [UUID] = []

    init(usersController: UsersController) {
        self.usersController = usersController
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        // setup collectionView + dataSource here...

        usersController.willChange = { [weak self] in
            guard let self else { return }
            pendingSelectedIDs =
                (collectionView.indexPathsForSelectedItems ?? [])
                    .compactMap { dataSource.itemIdentifier(for: $0) }
        }

        usersController.didChange = { [weak self] in
            self?.reloadSnapshot(preserveSelection: true)
        }

        Task { await usersController.load() }
    }

    private func reloadSnapshot(animated: Bool = true, preserveSelection: Bool) {
        var snapshot =
            NSDiffableDataSourceSnapshot<UsersController.SectionID, UUID>()

        snapshot.appendSections(usersController.users.map(\.id))
        for section in usersController.users {
            snapshot.appendItems(section.users.map(\.id), toSection: section.id)
        }

        dataSource.apply(snapshot, animatingDifferences: animated) { [weak self] in
            guard let self, preserveSelection else { return }
            for id in pendingSelectedIDs {
                if let ip = dataSource.indexPath(for: id) {
                    collectionView.selectItem(at: ip, animated: false, scrollPosition: [])
                }
            }
            pendingSelectedIDs.removeAll()
        }
    }
}
```

### AppKit example (NSCollectionView; NSTableView is analogous)

```swift
@MainActor
final class UsersViewController: NSViewController {

    private let usersController: UsersController

    private let collectionView = NSCollectionView()
    private var dataSource: NSCollectionViewDiffableDataSource<UsersController.SectionID, UUID>!

    private var pendingSelectedIDs: [UUID] = []

    init(usersController: UsersController) {
        self.usersController = usersController
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        // setup collectionView + dataSource here...

        usersController.willChange = { [weak self] in
            guard let self else { return }
            pendingSelectedIDs =
                collectionView.selectionIndexPaths
                    .compactMap { dataSource.itemIdentifier(for: $0) }
        }

        usersController.didChange = { [weak self] in
            self?.reloadSnapshot(preserveSelection: true)
        }

        Task { await usersController.load() }
    }

    private func reloadSnapshot(animated: Bool = true, preserveSelection: Bool) {
        var snapshot =
            NSDiffableDataSourceSnapshot<UsersController.SectionID, UUID>()

        snapshot.appendSections(usersController.users.map(\.id))
        for section in usersController.users {
            snapshot.appendItems(section.users.map(\.id), toSection: section.id)
        }

        dataSource.apply(snapshot, animatingDifferences: animated) { [weak self] in
            guard let self, preserveSelection else { return }
            let paths = Set(pendingSelectedIDs.compactMap { dataSource.indexPath(for: $0) })
            collectionView.selectionIndexPaths = paths
            pendingSelectedIDs.removeAll()
        }
    }
}
```

## When to Add Helpers or Stores
Helpers/abstractions are allowed only if they pay rent:
- Multiple screens repeat the exact same wiring.
- Snapshot logic becomes unmanageable.
- Shared, testable snapshot-building is needed.

Default is no helper frameworks. Keep snapshot logic in the view controller and side controller.

## Static Tables (Structural Data Only)
Static tables may be described using simple structural data types. These types describe structure only.

Allowed pattern:
- TableData
- TableSection
- Cell (enum)

Requirements:
- Structural only. No business logic, UI logic, callbacks, or state.
- Enums define identity and intent.
- Structs define grouping and ordering.

Example:

```swift
enum SettingsCell: Hashable {
    case displayName
    case marketingEmails
    case signOut
}

struct TableSection<CellID: Hashable> {
    let header: String?
    let footer: String?
    let cells: [CellID]
}

struct TableData<CellID: Hashable> {
    let sections: [TableSection<CellID>]
}
```

Usage:
- Side controllers may derive TableData from their state.
- View controllers render tables by switching on the Cell enum.
- All behavior (taps, toggles, text changes) lives in the view controller
  or is delegated back to the side controller.

Not allowed:
- No closures inside table data.
- No per-cell mutable state.
- No layout or formatting logic.
- No UIKit/AppKit types in these structs.

This pattern is intended for static or mostly-static tables
(e.g. Settings, Preferences, Info screens).
