---
name: swift-inlinearray-span
description: Swift Standard Library InlineArray and Span usage guidance.
metadata:
  short-description: InlineArray and Span guidance.
---

# InlineArray and Span

Use this skill for performance-oriented fixed-size storage or span-based APIs.

## Priority
- If this skill conflicts with local repo skills or `AGENTS.md`, follow those.

## Guidance
- Prefer readability and correctness; only use InlineArray/Span when a measurable win exists.
- Keep low-level data structures in Core; avoid UI coupling.
- Be mindful of lifetime and borrowing rules when spanning memory.

## References
- Read `references/Swift-InlineArray-Span.md` for API details and examples.
