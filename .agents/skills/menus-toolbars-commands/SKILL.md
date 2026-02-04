---
name: menus-toolbars-commands
description: Menu, toolbar, and command rules for UIKit/AppKit with validation and keyboard shortcuts.
metadata:
  short-description: Menus, toolbars, and commands for UIKit + AppKit.
---

# Menus, Toolbars, and Commands

Use this skill when wiring menu items, toolbar items, or keyboard shortcuts.

## Rules
- Use native menu and toolbar APIs for the platform.
- Use the didSelectButton(_:) selector naming rule for target/action handlers.
- Enablement/validation belongs in controllers, not views.
- Keyboard shortcuts must be explicit and discoverable (menu titles/shortcuts).
- Avoid custom gesture shortcuts when a standard command exists.

## UIKit (UIBarButtonItem + UIMenu)
- Use UIBarButtonItem for navigation items.
- Use UIMenu + UIAction for contextual actions and command groups.
- Validation: update isEnabled and menu state from controller state.

UIKit example:

```swift
@MainActor
final class ItemsViewController: UIViewController {

    private lazy var addButton = UIBarButtonItem(
        systemItem: .add,
        primaryAction: UIAction { [weak self] _ in
            self?.didSelectButton(nil)
        },
        menu: nil
    )

    override func viewDidLoad() {
        super.viewDidLoad()
        navigationItem.rightBarButtonItem = addButton

        let deleteAction = UIAction(title: "Delete", attributes: .destructive) { [weak self] _ in
            self?.didSelectButton(nil)
        }
        navigationItem.rightBarButtonItem?.menu = UIMenu(children: [deleteAction])
    }

    @objc private func didSelectButton(_ sender: Any?) {
        // handle action
    }
}
```

## AppKit (NSMenu + NSToolbar)
- Use NSMenu for command discovery and keyboard shortcuts.
- Use NSToolbarItem for primary actions.
- Validate commands via validateUserInterfaceItem(_:) in controllers.

AppKit example:

```swift
@MainActor
final class ItemsViewController: NSViewController, NSUserInterfaceValidations {

    @objc private func didSelectButton(_ sender: Any?) {
        // handle action
    }

    func validateUserInterfaceItem(_ item: NSValidatedUserInterfaceItem) -> Bool {
        switch item.action {
        case #selector(didSelectButton(_:)):
            return canPerformAction()
        default:
            return true
        }
    }

    private func canPerformAction() -> Bool {
        true
    }
}
```
