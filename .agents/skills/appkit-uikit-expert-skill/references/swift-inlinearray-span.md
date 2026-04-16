# InlineArray and Span

Use this reference for performance-oriented fixed-size storage or span-based APIs.

## Priority
- If this reference conflicts with local repo guidance or `AGENTS.md`, follow those.

## Guidance
- Prefer readability and correctness; only use InlineArray/Span when a measurable win exists.
- Keep low-level data structures in Core; avoid UI coupling.
- Be mindful of lifetime and borrowing rules when spanning memory.

## References
- Read `references/Swift-InlineArray-Span.md` for API details and examples.
