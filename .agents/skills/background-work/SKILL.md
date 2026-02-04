---
name: background-work
description: BGTaskScheduler and AppKit background work patterns with threading and OSLog rules.
metadata:
  short-description: Background work rules for iOS + macOS.
---

# Background Work

Use this skill when adding or modifying background work on iOS or macOS.

## Rules
- Background work lives in side controllers/services, not view controllers.
- Never block the main thread. UI updates must happen on the main thread (@MainActor).
- Use OSLog/Logger with subsystem + category. Do not log secrets or PII.
- Use privacy annotations for user data in logs.
- Work must be cancellable and respond to Task cancellation.
- iOS: declare task identifiers in Info.plist (BGTaskSchedulerPermittedIdentifiers).
- iOS: always reschedule the next task after completion.

## iOS (BGTaskScheduler)
- Register tasks in the iOS AppDelegate (not SceneDelegate).
- Use BGAppRefreshTask for short, frequent work.
- Use BGProcessingTask for longer work or when requiring constraints.
- Always set an expiration handler.
- If using background URLSession, use a background session identifier and hand off to a side controller.

AppDelegate registration example:

```swift
final class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.example.app.refresh",
            using: nil
        ) { task in
            guard let task = task as? BGAppRefreshTask else { return }
            BackgroundWorkController.shared.handleAppRefresh(task)
        }

        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.example.app.process",
            using: nil
        ) { task in
            guard let task = task as? BGProcessingTask else { return }
            BackgroundWorkController.shared.handleProcessing(task)
        }
        return true
    }
}
```

Side controller handling example:

```swift
final class BackgroundWorkController {

    static let shared = BackgroundWorkController()

    private let logger = Logger(subsystem: "App", category: "BackgroundWork")

    func handleAppRefresh(_ task: BGAppRefreshTask) {
        scheduleAppRefresh()
        task.expirationHandler = { [weak self] in
            self?.logger.error("App refresh expired")
        }
        Task {
            let success = await refresh()
            task.setTaskCompleted(success: success)
        }
    }

    func handleProcessing(_ task: BGProcessingTask) {
        scheduleProcessing()
        task.expirationHandler = { [weak self] in
            self?.logger.error("Processing expired")
        }
        Task {
            let success = await process()
            task.setTaskCompleted(success: success)
        }
    }

    func scheduleAppRefresh() {
        let request = BGAppRefreshTaskRequest(identifier: "com.example.app.refresh")
        request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)
        try? BGTaskScheduler.shared.submit(request)
    }

    func scheduleProcessing() {
        let request = BGProcessingTaskRequest(identifier: "com.example.app.process")
        request.requiresNetworkConnectivity = true
        request.requiresExternalPower = false
        try? BGTaskScheduler.shared.submit(request)
    }

    private func refresh() async -> Bool {
        // network/cache updates
        return true
    }

    private func process() async -> Bool {
        // longer background work
        return true
    }
}
```

Background URLSession handoff (pattern):
- Create a background session with an identifier.
- AppDelegate receives the completion handler and passes it to a side controller.
- Side controller finishes background events after tasks are complete.

## macOS (NSBackgroundActivityScheduler)
- Use NSBackgroundActivityScheduler for periodic background work.
- Configure interval and tolerance for system efficiency.
- Cancel the scheduler when work is no longer needed.

Example:

```swift
final class BackgroundWorkController {

    private let logger = Logger(subsystem: "App", category: "BackgroundWork")
    private let scheduler = NSBackgroundActivityScheduler(identifier: "com.example.app.refresh")

    func start() {
        scheduler.repeats = true
        scheduler.interval = 15 * 60
        scheduler.tolerance = 5 * 60
        scheduler.schedule { [weak self] completion in
            guard let self else {
                completion(.finished)
                return
            }
            Task {
                let success = await self.refresh()
                if !success {
                    self.logger.error("Background refresh failed")
                }
                completion(.finished)
            }
        }
    }

    func stop() {
        scheduler.invalidate()
    }

    private func refresh() async -> Bool {
        // fetch/cache updates
        return true
    }
}
```

## Logging and Threading
- Use Logger with privacy annotations for user data:

```swift
logger.info("Synced user: \(userID, privacy: .private)")
```

- UI updates must be performed on @MainActor.
