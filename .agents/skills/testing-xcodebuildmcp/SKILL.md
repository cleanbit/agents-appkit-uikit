---
name: testing-xcodebuildmcp
description: Testing requirements and XcodeBuildMCP-only build/test workflow for unit and UI tests.
metadata:
  short-description: Testing requirements and XcodeBuildMCP workflow.
---

# Testing and XcodeBuildMCP

Use this skill whenever code changes are made.

## Tests Are Required
- Changes must include both unit and UI tests.
- Unit tests must cover:
  - Side controllers (loaders, managers)
  - Data shaping (grouping, sectioning, sorting)
  - Core logic in the Core framework target
- Tests must be deterministic:
  - No real network
  - No real file system
  - No reliance on current time without injection
- UI tests are required for every change.
- Do not add fragile screenshot-based tests unless explicitly requested.

## Running Tests
- Tests must be run after every code change.
- All build/run/test actions must use XcodeBuildMCP (do not invoke xcodebuild directly).
- If tests are not feasible, stop and ask for explicit written approval before proceeding.

## Pull Request Hygiene
- Keep PRs small and focused.
- Update or add tests for any logic change.
- Ensure the project builds for all supported platforms.
- If SwiftLint is installed, it must run clean (no warnings or errors) before committing.

## Definition of Done
A change is done when:
- It compiles for the target platform.
- MVC roles are respected (logic on the side).
- List updates are snapshot-driven and centralized.
- Selection/focus behavior is preserved.
- Accessibility is not worse than before.
