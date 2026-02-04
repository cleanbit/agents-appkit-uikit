---
name: foundation-models-on-device-llm
description: Guidance for Foundation Models on-device LLM usage with UIKit/AppKit apps.
metadata:
  short-description: On-device Foundation Models guidance.
---

# Foundation Models: On-Device LLM

Use this skill when integrating Apple's Foundation Models or on-device LLM features.

## Priority
- If this skill conflicts with local repo skills or `AGENTS.md`, follow those.

## Guidance
- Keep model interactions in Core services; UI should consume results on the main thread.
- Treat prompts/outputs as sensitive; do not log or persist without explicit product requirements.
- Handle cancellation, timeouts, and streaming updates to avoid blocking UI.
- SwiftUI is forbidden; ignore any SwiftUI notes in references.

## References
- Read `references/FoundationModels-Using-on-device-LLM-in-your-app.md` for API details and patterns.
