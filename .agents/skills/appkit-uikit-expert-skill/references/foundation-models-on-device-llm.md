# Foundation Models: On-Device LLM

Use this reference when integrating Apple's Foundation Models or on-device LLM features.

## Priority
- If this reference conflicts with local repo guidance or `AGENTS.md`, follow those.

## Guidance
- Keep model interactions in Core services; UI should consume results on the main thread.
- Treat prompts/outputs as sensitive; do not log or persist without explicit product requirements.
- Handle cancellation, timeouts, and streaming updates to avoid blocking UI.
- SwiftUI is forbidden; ignore any SwiftUI notes in references.

## References
- Read `references/FoundationModels-Using-on-device-LLM-in-your-app.md` for API details and patterns.
