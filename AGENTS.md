# agents.md

This repository builds **iOS (UIKit)** and **macOS (AppKit)** apps in **Swift** using **MVC**.

**Controllers coordinate. Logic lives on the side.**
- View/ViewController: UI setup, event wiring, snapshot application, navigation.
- Side controllers/services: loading, mutation, grouping/sectioning, business rules.
- No architecture crusades. No mandatory view models.

---

## Non‑negotiables

- **Match local patterns** in adjacent code.
- **Keep diffs small** and scoped to the task.
- **Main-thread UI**: UI updates happen on the main thread (`@MainActor` where appropriate).
- **Accessibility**: don’t regress labels, keyboard navigation (macOS), selection/focus behavior.
- **No drive‑by refactors** unless explicitly requested.

---

## Swift conventions

- Prefer `let` over `var`.
- Prefer `private` by default; widen only when needed.
- Use `final` unless subclassing is intended.
- Use typed errors (`enum`) for domain failures; don’t silently swallow errors.
- Prefer Swift Concurrency (`async/await`) for new code.

---

## Lists & diffable data sources

We use diffable data sources on both platforms:

- iOS: `UITableViewDiffableDataSource`, `UICollectionViewDiffableDataSource`
- macOS: `NSTableViewDiffableDataSource`, `NSCollectionViewDiffableDataSource`

**Diffable is snapshot-driven.**
Something changes → rebuild snapshot → apply snapshot.

### Identity rules (critical)
- Item identifiers **must be stable** (`UUID`, database primary key, `NSManagedObjectID`, etc.).
- Never use array indices as identifiers.
- Preserve selection/focus by **IDs**, not index paths/rows.

---

## Side controllers (logic on the side)

Side controllers own the canonical data and notify the UI about changes. They expose:

- `willChange`: called **before** mutating the data
- `didChange`: called **after** data is updated
- a sectioned property (ordered)

### Canonical section model (recommended)
Use ordered sections to avoid dictionary-order surprises:

```swift
@MainActor
final class UsersController {

    enum SectionID: Hashable { case online, offline }

    struct User: Hashable {
        let id: UUID
        let name: String
        let isOnline: Bool
    }

    struct Section {
        let id: SectionID
        let users: [User]
    }

    private(set) var users: [Section] = []   // property may include sections

    var willChange: (() -> Void)?
    var didChange: (() -> Void)?

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

**Rule:** grouping/sectioning belongs in the side controller (or its helpers), not in the view controller.

---

## View controllers: snapshot application (canonical)

View controllers:
- configure table/collection + diffable data source
- capture selection during `willChange`
- rebuild/apply snapshot during `didChange`
- restore selection after apply

**One place** per screen for snapshot logic: `reloadSnapshot()`.

### UIKit example (collection view; tables are the same snapshot shape)

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

---

## Core Data: NSFetchedResultsController (FRC) + diffable

Core Data lists use **`NSFetchedResultsController`** for change notifications, then apply a snapshot.

### Rules
- FRC is the *change source*; diffable is the *UI updater*.
- Prefer rebuilding the snapshot on FRC changes (simple + correct).
- Use `NSManagedObjectID` as the diffable item identifier.
- Preserve selection by `NSManagedObjectID`.

### UIKit table example (FRC → snapshot)

```swift
final class MessagesViewController: UITableViewController, NSFetchedResultsControllerDelegate {

    enum SectionID: Hashable { case main } // or use frc.sections for multiple sections

    private var dataSource: UITableViewDiffableDataSource<SectionID, NSManagedObjectID>!
    private var frc: NSFetchedResultsController<Message>!

    private var pendingSelectedIDs: [NSManagedObjectID] = []

    override func viewDidLoad() {
        super.viewDidLoad()
        // configure table + dataSource here...

        frc.delegate = self
        try? frc.performFetch()
        reloadFromFRC()
    }

    func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        pendingSelectedIDs = tableView.indexPathsForSelectedRows?
            .compactMap { dataSource.itemIdentifier(for: $0) } ?? []
    }

    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        reloadFromFRC(preserveSelection: true)
    }

    private func reloadFromFRC(animated: Bool = true, preserveSelection: Bool = false) {
        let objects = frc.fetchedObjects ?? []

        var snapshot = NSDiffableDataSourceSnapshot<SectionID, NSManagedObjectID>()
        snapshot.appendSections([.main])
        snapshot.appendItems(objects.map { $0.objectID }, toSection: .main)

        dataSource.apply(snapshot, animatingDifferences: animated) { [weak self] in
            guard let self, preserveSelection else { return }
            for id in pendingSelectedIDs {
                if let ip = dataSource.indexPath(for: id) {
                    tableView.selectRow(at: ip, animated: false, scrollPosition: .none)
                }
            }
            pendingSelectedIDs.removeAll()
        }
    }
}
```

### Notes on sectioned FRC
If you use `sectionNameKeyPath`, you can:
- use section identifiers derived from `frc.sections?.map { $0.name }`
- append items per section in the same order
- keep identifiers stable (section name strings are typically stable enough)

---

## When to add helpers or stores
Helpers/abstractions are allowed only if they pay rent:
- multiple screens repeat the exact same wiring
- snapshot logic becomes unmanageable
- shared, testable snapshot-building is needed

Default is **no helper frameworks**: keep it in the view controller + side controller.

---


---

## SwiftUI policy (explicit)

This repository **does not use SwiftUI**.

- ❌ Do not introduce SwiftUI views, modifiers, or hosting controllers.
- ❌ Do not mix SwiftUI into UIKit or AppKit screens.
- ❌ Do not propose SwiftUI refactors or hybrid approaches.

All UI must be built with:
- **UIKit** on iOS
- **AppKit** on macOS

This is a deliberate architectural choice. Respect it.

---

## Shared / reusable code (SPM requirement)

All **shareable, non‑UI code** must live in a **separate Swift Package Manager (SPM) module**.

### What goes into the shared SPM package
- domain models
- business logic
- networking clients
- persistence layers
- Core Data stacks (model + helpers)
- parsing / formatting
- side controllers (e.g. `UsersController`)
- utilities that are platform‑agnostic

### What must NOT go into the shared package
- UIKit code
- AppKit code
- view controllers
- views, cells, layout code
- platform‑specific behavior

### Rules
- UI targets depend on the shared SPM package.
- The shared package must not depend on UIKit or AppKit.
- Conditional compilation (`#if os(iOS)`) inside the shared package is discouraged and must be justified.

This keeps core logic testable, reusable, and independent of UI frameworks.



---

---

## Project structure and naming

### Structure
- Use a consistent project structure with folder layout determined by **app features** (feature-first), not by file type.
- Keep iOS and macOS app targets separate, and keep shared logic in the Core SPM package.

Suggested high-level layout:

- `Apps/MyProduct-UIKit/<FeatureName>/...`
- `Apps/MyProduct-AppKit/<FeatureName>/...`
- `Packages/Core/Sources/<FeatureName>/...`
- `Packages/Core/Tests/<FeatureName>/Tests/...`

### Naming conventions (strict)
- Follow strict naming conventions for:
  - types (`PascalCase`)
  - properties/methods (`lowerCamelCase`)
  - files: match the primary type name in the file (`UserCell.swift` contains `UserCell`)
  - SwiftData/Core Data models: use clear, singular nouns (`User`, `Message`, not `Users`)
- Avoid abbreviations unless they are common domain terms.

### File hygiene
- Prefer **one primary type per file**.
- Split different structs/classes/enums into separate Swift files instead of stacking many unrelated types in one file.
- Use `// MARK:` to organize large types when needed.

### Comments and documentation
- Add comments where intent is non-obvious or when a decision has tradeoffs.
- Use doc comments (`///`) for public APIs, Core package types, and non-trivial helpers.
- Do not comment obvious code.

### Secrets
- Never commit secrets (API keys, tokens, credentials) into the repository.
- Use configuration files excluded from git, build settings, environment variables, or the platform keychain as appropriate.


## SwiftUI is forbidden

This repository is **UIKit + AppKit only**.

- **Do not add SwiftUI views** (`SwiftUI.View`) anywhere.
- **Do not add `UIHostingController` / `NSHostingController`**.
- **Do not add SwiftUI previews**.
- If a feature seems easier in SwiftUI, implement it in UIKit/AppKit instead.

(Exceptions require an explicit, written request in the task/PR description.)

---

## Shared core must live in a separate Swift Package

All shareable, platform-agnostic code must live in a dedicated **Swift Package Manager (SPM)** module (the “Core” package), not inside app targets.

### Put in Core (SPM)
- domain models (pure Swift structs/enums)
- services (protocols + implementations that are platform-agnostic)
- networking/persistence logic that does not depend on UIKit/AppKit
- parsing/formatting utilities
- state/transformation functions

### Keep out of Core
- UIKit/AppKit types (`UIView`, `NSView`, `UIViewController`, `NSViewController`, etc.)
- platform UX behaviors (menus, windows, clipboard, drag/drop)
- any UI layer code

### Rules for agents
- If you write code that is used by both iOS and macOS, it **must** go into the Core SPM package.
- If you add a new cross-platform type, add it to Core first and import it from the app targets.
- Do not introduce “shared UI” abstractions to force UIKit/AppKit into one API.


---

## Tests are required

Changes must include appropriate tests.

### Required
- **Unit tests** for:
  - side controllers (loaders, managers)
  - data shaping (grouping, sectioning, sorting)
  - Core logic in the shared Core SPM package
- Tests must be **deterministic**:
  - no real network
  - no real file system
  - no reliance on current time without injection

### UI tests
- Add UI tests only for:
  - critical user flows
  - regressions that cannot be covered by unit tests
- Do not add fragile screenshot-based tests unless explicitly requested.

### Rules for agents
- If logic is added or changed, tests must be added or updated.
- If tests are not feasible, the PR must explicitly explain why.

---

## Liquid Glass (mandatory for UI work)

This project adopts **Liquid Glass** design principles.

### Requirements
- Prefer system materials (`UIBlurEffect`, `NSVisualEffectView`) where appropriate.
- Respect translucency, depth, and layering.
- Avoid hard, opaque backgrounds unless required for readability.
- Use dynamic system colors; avoid hard-coded colors.
- Motion and transitions should be subtle and platform-appropriate.

### UIKit
- Use system-provided materials and vibrancy where applicable.
- Avoid custom blur implementations.

### AppKit
- Prefer `NSVisualEffectView` for backgrounds and containers.
- Ensure vibrancy works correctly with text and selection.

### Rules for agents
- UI changes must not break Liquid Glass aesthetics.
- If a design deviates from Liquid Glass, the reason must be explicit.

---

## Project structure & conventions

### Structure
- Use a **feature-based folder structure** (by screen/feature), not by technical layer.
- Keep UIKit/AppKit code inside the app targets only.
- Keep shareable, platform-agnostic code in the **Core SPM package**.

### Files
- **One primary type per Swift file**.
  - Do not place multiple unrelated structs/classes/enums in a single file.
- File name must match the primary type name.

### Naming
- Follow strict, consistent naming conventions:
  - Types: `UpperCamelCase`
  - Methods & properties: `lowerCamelCase`
  - Avoid abbreviations unless they are well-known.
- SwiftData / Core Data models must use clear, explicit names (no generic `Item`, `Data`, etc.).

### Documentation & comments
- Add code comments where intent is not obvious.
- Use documentation comments (`///`) for public APIs and non-trivial logic.
- Do not comment obvious code.

### Secrets
- **Never commit secrets** (API keys, tokens, credentials).
- Use environment variables, configuration files ignored by git, or secure storage as appropriate.

---

## Testing rules (reinforced)

- Unit tests are the default and expected.
- UI tests should only be added when unit tests are not feasible.
- Core logic must always be unit-tested (especially code in the Core SPM package).

---

## Pull request instructions

- Keep PRs small and focused.
- Update or add tests for any logic change.
- Ensure the project builds for all supported platforms.
- **If SwiftLint is installed, it must run clean** (no warnings or errors) before committing.

---

## Number and value formatting

- **Never use C-style formatting** such as `String(format:)` for numbers.
- Always use **locale-aware Foundation formatting**.

### Preferred approaches
- Use `FormatStyle`:
  ```swift
  let text = value.formatted(.number.precision(.fractionLength(2)))
  ```
- Use `NumberFormatter` for reusable or more complex formatting needs.

### Notes
- This rule applies to **UIKit**, **AppKit**, and **shared Core (SPM)** code.
- SwiftUI examples of formatting may appear in external documentation, but **SwiftUI itself is forbidden** in this repository.

---

## Concurrency and timing

- **Never use** `Task.sleep(nanoseconds:)`.
- **Always use** `Task.sleep(for:)` with `Duration`.

### Rationale
- Improves readability and correctness.
- Avoids manual nanosecond calculations.
- Aligns with modern Swift Concurrency APIs.

### Example
```swift
try await Task.sleep(for: .seconds(1))
```

This rule applies to UIKit, AppKit, and shared Core (SPM) code.

---

## Modern Swift and Foundation usage

### Concurrency assumptions
- Assume **strict Swift concurrency** rules are enabled.
- Respect actor isolation and `Sendable` requirements.
- Do not silence concurrency warnings without a clear, documented reason.

### Prefer Swift-native APIs
- Prefer Swift-native alternatives where they exist.
  - Example:
    ```swift
    let result = string.replacing("hello", with: "world")
    ```
    instead of:
    ```swift
    let result = string.replacingOccurrences(of: "hello", with: "world")
    ```

### Prefer modern Foundation APIs
- Use modern, strongly-typed Foundation conveniences:
  - `URL.documentsDirectory` instead of manual directory lookups
  - `url.appending(path:)` instead of string-based path concatenation
- Avoid legacy APIs that rely on stringly-typed paths or implicit assumptions.

These rules apply to UIKit, AppKit, and shared Core (SPM) code.

---


---

## Layout and Auto Layout (Final)

All UI layout must be implemented using **Auto Layout** with **layout anchors**.  
Frame-based layout and autoresizing masks are not allowed for production UI.

This applies to **UIKit and AppKit**.

### Auto Layout requirements
- Use **Auto Layout exclusively** for layout.
- Prefer **layout anchors** (`NSLayoutAnchor` APIs).
- Do **not** rely on autoresizing masks (`translatesAutoresizingMaskIntoConstraints = false`).

### Layout margins (default)
**Layout margins are the default constraint reference.**

- Prefer `layoutMarginsGuide` over raw view edges.
- Use layout margins for leading/trailing/top/bottom constraints and general content alignment.

UIKit:
```swift
view.layoutMarginsGuide.leadingAnchor
view.layoutMarginsGuide.trailingAnchor
view.layoutMarginsGuide.topAnchor
view.layoutMarginsGuide.bottomAnchor
```

AppKit:
```swift
view.layoutMarginsGuide.leadingAnchor
view.layoutMarginsGuide.trailingAnchor
view.layoutMarginsGuide.topAnchor
view.layoutMarginsGuide.bottomAnchor
```

### Hard-coded spacing
- Avoid hard-coded spacing constants (`8`, `12`, `16`, etc.).
- Prefer layout margins, intrinsic content size, and system spacing relations.
- Hard-coded constants are allowed **only when explicitly required by design**.

### Valid exceptions
Raw anchors are allowed when:
- aligning sibling views
- building custom controls
- drawing edge-to-edge backgrounds
- implementing separators
- performance-critical measurement-only layout

### Constraint hygiene
- Keep constraints close to the views they lay out.
- Group and activate constraints together.
- Readability is required.

### Platform notes
- **UIKit**: Auto Layout + anchors + margins only.
- **AppKit**: Auto Layout + anchors + margins only; avoid springs-and-struts.

> **Default rule:** Auto Layout + anchors + layout margins. Deviate only with intent.

---

## Static tables (structural data only)

Static tables may be described using **simple structural data types**.
These types exist only to describe *structure*, not behavior.

### Allowed pattern

Use a small hierarchy like:

- `TableData`
- `TableSection`
- `Cell` (enum)

### Requirements
- These types must be **structural only**.
- Do **not** put business logic, UI logic, callbacks, or state inside them.
- Enums define **identity and intent**.
- Structs define **grouping and ordering**.

### Example

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

### Usage
- Side controllers may **derive** `TableData` from their state.
- View controllers render tables by switching on the `Cell` enum.
- All behavior (taps, toggles, text changes) lives in the view controller
  or is delegated back to the side controller.

### Not allowed
- No closures inside table data.
- No per-cell mutable state.
- No layout or formatting logic.
- No UIKit/AppKit types in these structs.

This pattern is intended for **static or mostly-static tables**
(e.g. Settings, Preferences, Info screens).

## Definition of done
A change is done when:
- it compiles for the target platform
- MVC roles are respected (logic on the side)
- list updates are snapshot-driven and centralized
- selection/focus behavior is preserved
- accessibility is not worse than before
