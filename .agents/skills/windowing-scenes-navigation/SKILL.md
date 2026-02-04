---
name: windowing-scenes-navigation
description: UIScene lifecycle, multiwindow, NSSplitViewController/NSWindowController navigation, and state restoration.
metadata:
  short-description: Windowing, scenes, and navigation rules.
---

# Windowing, Scenes, and Navigation

Use this skill when wiring scene/window lifecycle, multiwindow behavior, or primary navigation.

## Rules
- iOS uses UIScene. Do not drive lifecycle solely from UIApplicationDelegate.
- macOS windows are owned by NSWindowController.
- Window/scene setup belongs in SceneDelegate (iOS) and NSWindowController (macOS).
- Side controllers remain UI-agnostic; UI state is coordinated by controllers.
- State restoration uses NSUserActivity with minimal, typed state.
- Avoid global singletons for window state; keep per-window state in controllers.

## iOS (UIScene + Multiwindow)
- Build the window and root controller in scene(_:willConnectTo:options:).
- Support multiple windows by using scene sessions and user activities.
- Use user activity identifiers to route to the correct root/selection state.

Example:

```swift
final class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    var window: UIWindow?

    func scene(
        _ scene: UIScene,
        willConnectTo session: UISceneSession,
        options connectionOptions: UIScene.ConnectionOptions
    ) {
        guard let windowScene = scene as? UIWindowScene else { return }
        let window = UIWindow(windowScene: windowScene)
        window.rootViewController = RootViewController()
        window.makeKeyAndVisible()
        self.window = window

        if let activity = connectionOptions.userActivities.first {
            restore(activity)
        }
    }

    func sceneDidEnterBackground(_ scene: UIScene) {
        // delegate persistence to side controllers
    }

    private func restore(_ activity: NSUserActivity) {
        // read identifiers from activity.userInfo
    }
}
```

Creating a new window with user activity:

```swift
let activity = NSUserActivity(activityType: "com.example.app.window")
activity.userInfo = ["route": "detail", "id": userID.uuidString]
UIApplication.shared.requestSceneSessionActivation(
    nil,
    userActivity: activity,
    options: nil
)
```

## macOS (NSWindowController + NSSplitViewController)
- AppDelegate owns the initial window controller and shows it.
- NSWindowController configures the root split view controller.
- Use NSSplitViewController for primary navigation when applicable.

Example:

```swift
final class MainWindowController: NSWindowController {

    override func windowDidLoad() {
        super.windowDidLoad()
        let split = NSSplitViewController()
        split.addSplitViewItem(NSSplitViewItem(sidebarWithViewController: SidebarViewController()))
        split.addSplitViewItem(NSSplitViewItem(viewController: DetailViewController()))
        contentViewController = split
    }
}
```

## State Restoration (Basic)
- Use NSUserActivity for lightweight state (selected ID, route, window role).
- Keep keys and values stable and typed.
- Restore in SceneDelegate (iOS) and NSWindowController (macOS).

Example keys:

```swift
enum RestorationKeys {
    static let route = "route"
    static let selectedID = "selectedID"
}
```
