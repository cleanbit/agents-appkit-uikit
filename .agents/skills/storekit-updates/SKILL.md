---
name: storekit-updates
description: StoreKit updates and in-app purchase integration guidance for UIKit/AppKit apps.
metadata:
  short-description: StoreKit updates and IAP guidance.
---

# StoreKit Updates

Use this skill when implementing or updating StoreKit functionality.

## Priority
- If this skill conflicts with local repo skills or `AGENTS.md`, follow those.

## Guidance
- Keep purchase/receipt logic in services or side controllers; UI handles presentation only.
- Use StoreKit testing tools and environments; never log receipts or user identifiers.
- SwiftUI StoreKit views are not applicable; ignore SwiftUI sections in references.

## References
- Read `references/StoreKit-Updates.md` for updated APIs and testing guidance.
