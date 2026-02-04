---
name: mapkit-place-descriptors
description: MapKit GeoToolbox place descriptor usage and integration guidance.
metadata:
  short-description: MapKit place descriptors.
---

# MapKit Place Descriptors

Use this skill when working with place descriptors and GeoToolbox APIs.

## Priority
- If this skill conflicts with local repo skills or `AGENTS.md`, follow those.

## Guidance
- Keep geocoding and descriptor resolution in Core services; UI should render outputs.
- Handle errors and rate limits gracefully; cache results where appropriate.
- Use locale-aware formatting for place display strings.

## References
- Read `references/MapKit-GeoToolbox-PlaceDescriptors.md` for API usage and patterns.
