# agents.md

This repository builds **iOS (UIKit)** and **macOS (AppKit)** apps in **Swift** using **MVC**.
iOS **always** includes iPadOS throughout these instructions.

**Required:** The project may use a single multi‑destination target **or** separate iOS and macOS targets, depending on the repo layout. Follow the project’s existing target structure.

### Target membership (separate targets)
This project uses separate iOS and macOS targets. Target membership is the primary gate for
platform-specific files.

Rules:
- All files under `Code/iOS/**` must be members of the iOS target only.
- All files under `Code/macOS/**` must be members of the macOS target only.
- All shared, non-UI files under `Code/Core/**` must be members of the Core framework target.
- Duplicate filenames across iOS/macOS are allowed only when membership excludes the other platform.
- Do not rely on `#if os(iOS)` / `#if os(macOS)` alone for platform-only files; target membership is required.

Example membership:
- iOS target: `Code/iOS/Root/RootSplitViewController.swift`
- macOS target: `Code/macOS/Root/RootSplitViewController.swift`
- Core target: `Code/Core/Users/UsersController.swift`

### Platform entry points & platform-only files

- **macOS must use `MainMenu.xib`**; keep it in the macOS target and do not remove it.
- **iOS must use UIScene** (`UISceneDelegate` / `SceneDelegate`); do not drive iOS lifecycle solely from `UIApplicationDelegate`.
- **AppDelegate files are per‑platform**: separate iOS and macOS `AppDelegate` files with target membership set to their platform.
- **Do not use `#if os()` for whole files**; only use small, localized branches when sharing a file is unavoidable.
- Target membership is the primary gate for platform‑specific files; do not rely on `#if os()` alone.

### Localization (xcstrings)

- Use `Resources/Localizable.xcstrings` as the single source of localized strings.
- Do not add `.strings` or `.stringsdict` files unless explicitly requested.

**Required:** Storyboards and XIBs are not allowed **except**:
- iOS: `Resources/iOS/Base.lproj/Main.storyboard` and `Resources/iOS/Base.lproj/LaunchScreen.storyboard`.
- macOS: `Resources/macOS/Base.lproj/MainMenu.xib` (must be kept and used).
All other UI must be created programmatically.

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
- **Xcode builds/runs**: All Xcode build/run actions must use **XcodeBuildMCP**; do not invoke `xcodebuild` directly.
- **No drive‑by refactors** unless explicitly requested.

---

## Most Native = Best UX

- Prefer platform-standard controls and behaviors; don’t fight UIKit/AppKit defaults.
- Keep navigation and command placement aligned with platform conventions.
- Respect keyboard shortcuts, focus rings, and selection behavior on macOS.
- Avoid “one-size-fits-all” UI that ignores platform idioms.

Examples:
- Use native context menus instead of custom popovers.
- Prefer standard toolbar items on macOS over bespoke header buttons.

---

## SF Symbols (required)

- Use SF Symbols for all icons on iOS and macOS.
- Do not add custom icon assets unless explicitly requested.
- Prefer system-provided symbols with platform-appropriate weights and scales.

---

## Logging

- Use `OSLog` / `Logger` for app logging; avoid `print` except for temporary local debugging.
- Prefer logging in side controllers/services; UI classes should log only UI lifecycle/navigation when needed.
- Always use subsystem + category; keep categories consistent and feature-scoped.
- Never log secrets or PII; use privacy annotations (`privacy: .private`) for user data.

Example:

```swift
import OSLog

private let logger = Logger(
    subsystem: Bundle.main.bundleIdentifier ?? "App",
    category: "Networking"
)

logger.info("Loaded \(count, privacy: .public) items")
```

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

### UsersController example (non-Core Data)
Use ordered sections to avoid dictionary-order surprises:

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
- If a controller’s underlying data is Core Data, it must use `NSFetchedResultsController` for change tracking. Keep the controller’s public API stable (e.g., `willChange`, `didChange`, sectioned data model) and avoid exposing FRC types to UI.
- FRC is the *change source*; diffable is the *UI updater*.
- Prefer rebuilding the snapshot on FRC changes (simple + correct).
- Use `NSManagedObjectID` as the diffable item identifier.
- Preserve selection by `NSManagedObjectID`.

### Active Record Pattern (Core Data Required)
- Core Data entities must expose Active Record helpers directly on the entity (class or extension).
- Every action must have two methods: one that accepts `NSManagedObjectContext`, and one that uses the main context.
- Context-free methods must delegate to the context-aware method using the main context.
- Main context must come from an `NSPersistentContainer` (for example, `container.viewContext`).
- Use typed errors (`enum`) for CRUD failures and wrap Core Data errors.
- Context-aware methods must use `context.perform` or `context.performAndWait` for queue confinement.

Example: main context creation with `NSPersistentContainer`

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

### FRC setup (uses Active Record main context)

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

Use entity fetch requests for FRC wiring. Active Record `fetch()` is for direct loads,
not for building an `NSFetchedResultsController`.

### UsersController example (Core Data via FRC)

```swift
@MainActor
final class UsersController: NSObject, NSFetchedResultsControllerDelegate {

    typealias ResultType = User

    private let frc: NSFetchedResultsController<User>

    var sections: [any NSFetchedResultsSectionInfo]? {
        frc.sections
    }

    var fetchedObjects: [ResultType]? {
        frc.fetchedObjects
    }

    private(set) var users: [Section] = []

    var willChange: (() -> Void)?
    var didChange: (() -> Void)?

    init(frc: NSFetchedResultsController<User>) {
        self.frc = frc
        super.init()
        self.frc.delegate = self
    }

    func load() throws {
        try frc.performFetch()
    }

    func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        willChange?()
    }

    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        didChange?()
    }

    func object(at indexPath: IndexPath) -> ResultType {
        frc.object(at: indexPath)
    }

    func indexPath(forObject object: ResultType) -> IndexPath? {
        frc.indexPath(forObject: object)
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
(Exceptions require an explicit, written request in the task/PR description.)

---

## Shared / reusable code (Core framework requirement)

All **shareable, non‑UI code** must live in the **Core framework target** under `Code/Core`.

### What goes into the Core framework target
- domain models (pure Swift structs/enums)
- services (protocols + implementations that are platform‑agnostic)
- business logic
- networking clients
- persistence layers
- Core Data stacks (model + helpers)
- parsing / formatting
- state/transformation functions
- side controllers (e.g. `UsersController`)
- utilities that are platform‑agnostic

### What must NOT go into the Core framework target
- UIKit code
- AppKit code
- view controllers
- views, cells, layout code
- platform‑specific behavior (menus, windows, clipboard, drag/drop)
- any UI layer code

### Rules
- UI targets depend on the Core framework target.
- The Core framework target must not depend on UIKit or AppKit.
- Conditional compilation (`#if os(iOS)`) inside the Core framework target is discouraged and must be justified.
- If you write code that is used by both iOS and macOS, it **must** go into the Core framework target.
- If you add a new cross-platform type, add it to Core first and import it from the app targets.
- Do not introduce “shared UI” abstractions to force UIKit/AppKit into one API.

This keeps core logic testable, reusable, and independent of UI frameworks.



---

---

## Project structure and naming

### Structure
- Use a consistent project structure with folder layout determined by **app features** (feature-first), not by file type.
- **Required**: Internal shared framework targets must be integrated as local, in-repo targets and edited from within the same Xcode project. Remote Git dependencies are reserved for third-party libraries only.

Layout (current project layout takes precedence):

- `Code/Core/<FeatureName>/...` for shared, platform-agnostic code
- `Code/iOS/<FeatureName>/...` for UIKit screens and iOS-specific code
- `Code/macOS/<FeatureName>/...` for AppKit screens and macOS-specific code
- `Resources/iOS/Base.lproj/Main.storyboard`
- `Resources/iOS/Base.lproj/LaunchScreen.storyboard`
- `Resources/macOS/Base.lproj/MainMenu.xib`

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
- Use doc comments (`///`) for public APIs, Core framework types, and non-trivial helpers.
- Do not comment obvious code.

### Secrets
- Never commit secrets (API keys, tokens, credentials) into the repository.
- Use configuration files excluded from git, build settings, environment variables, or the platform keychain as appropriate.


## Tests are required

Changes must include **both unit and UI tests**.

### Required
- **Unit tests** for:
  - side controllers (loaders, managers)
  - data shaping (grouping, sectioning, sorting)
  - Core logic in the shared Core framework target
- Tests must be **deterministic**:
  - no real network
  - no real file system
  - no reliance on current time without injection

### UI tests
- UI tests are required for every change.
- Do not add fragile screenshot-based tests unless explicitly requested.

### Rules for agents
- If logic is added or changed, tests (unit and UI) must be added or updated.
- Tests must be run after every code change.
- All test runs must use **XcodeBuildMCP**.
- If tests are not feasible, stop and ask for explicit written approval before proceeding.

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
- Keep shareable, platform-agnostic code in the **Core framework target**.

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

- Unit tests are required for all changes.
- UI tests are required for all changes.
- Tests must be updated and run after every code change using **XcodeBuildMCP**.
- Core logic must always be unit-tested (especially code in the Core framework target).

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
- This rule applies to **UIKit**, **AppKit**, and **shared Core framework** code.
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

This rule applies to UIKit, AppKit, and shared Core framework code.

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

These rules apply to UIKit, AppKit, and shared Core framework code.

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
