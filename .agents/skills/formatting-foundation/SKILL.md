---
name: formatting-foundation
description: Swift conventions plus locale-aware formatting and modern Foundation API guidance, including concurrency timing rules.
metadata:
  short-description: Swift conventions and locale-aware formatting with modern Foundation.
---

# Swift Conventions and Foundation Formatting

Use this skill for Swift coding conventions, numeric formatting, and modern Foundation usage.

## Swift Conventions
- Prefer let over var.
- Prefer private by default; widen only when needed.
- Use final unless subclassing is intended.
- Use typed errors (enum) for domain failures; do not silently swallow errors.
- Prefer Swift Concurrency (async/await) for new code.

## Number and Value Formatting
- Never use C-style formatting such as String(format:) for numbers.
- Always use locale-aware Foundation formatting.

Preferred approaches:
- FormatStyle:
  ```swift
  let text = value.formatted(.number.precision(.fractionLength(2)))
  ```
- NumberFormatter for reusable or more complex formatting needs.

Notes:
- This rule applies to UIKit, AppKit, and shared Core framework code.
- SwiftUI formatting examples may appear in external docs, but SwiftUI itself is forbidden in this repo.

## Concurrency and Timing
- Never use Task.sleep(nanoseconds:).
- Always use Task.sleep(for:) with Duration.

Example:
```swift
try await Task.sleep(for: .seconds(1))
```

## Modern Swift and Foundation Usage
Concurrency assumptions:
- Assume strict Swift concurrency rules are enabled.
- Respect actor isolation and Sendable requirements.
- Do not silence concurrency warnings without a clear, documented reason.

Prefer Swift-native APIs when they exist:
```swift
let result = string.replacing("hello", with: "world")
```
instead of:
```swift
let result = string.replacingOccurrences(of: "hello", with: "world")
```

Prefer modern Foundation APIs:
- URL.documentsDirectory instead of manual directory lookups
- url.appending(path:) instead of string-based path concatenation
- Avoid legacy APIs that rely on stringly-typed paths or implicit assumptions
