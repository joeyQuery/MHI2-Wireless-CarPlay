# Documentation Source of Truth

The repository contains several documentation layers. They have different authority and must not be treated as interchangeable.

## Authority order

When two documents appear to disagree, use this order:

1. **Completed trace artifacts** in `traces/`
2. **Evidence Register** in `docs/evidence.md`
3. **Subsystem research** in `docs/bluetooth.md`, `docs/iap2.md`, `docs/wifi.md`, `docs/airplay.md`, and `docs/carplay.md`
4. **Architecture synthesis** in `docs/wireless-carplay-architecture.md`
5. **STATUS.md**
6. **README.md**

## Important rule

Higher-level documents summarize lower-level evidence. They must not introduce a stronger claim than their source evidence supports.

A Mermaid arrow in an architecture document is not automatically a proven execution edge.

## Status vocabulary

Use consistently:

- **Proven** — directly supported by MHI2-specific evidence.
- **Partial** — the boundary/components are established, but the complete execution/data path is missing.
- **Target** — an investigation objective only.
- **Unproven** — plausible but unsupported by sufficient MHI2-specific evidence.
- **Disproven** — the previous interpretation has been eliminated or shown unsafe to assume.

## Evidence IDs

Every substantive trace conclusion should reference one or more IDs from `docs/evidence.md`.

## Provenance

A conclusion that depends on a particular firmware image must identify that image or explicitly state that provenance is incomplete. Never silently combine static evidence from one baseline with runtime evidence from another.

## Scope boundary

This document does not create runtime-capture or experiment repositories. Those remain outside the current documentation structure by project scope.
