# Swift Concurrency Updates

Use this reference when reviewing or updating concurrency behavior.

## Priority
- If this reference conflicts with local repo guidance or `AGENTS.md`, follow those.

## Guidance
- Enforce Sendable and actor isolation; avoid shared mutable global state.
- Keep UI work on `@MainActor`.
- Prefer structured concurrency and `Task.sleep(for:)` (see `formatting-foundation.md`).

## References
- Read `references/Swift-Concurrency-Updates.md` for the latest Swift 6.2 changes.
