# Documentation Index

The repository separates architecture, subsystem research, evidence, provenance and function-level tracing so that hypotheses do not become indistinguishable from verified behaviour.

## Start Here

1. [Repository Status](../STATUS.md)
2. [Wireless CarPlay Architecture](wireless-carplay-architecture.md)
3. [Roadmap](roadmap.md)
4. [Evidence Register](evidence.md)
5. [Disproven / Eliminated Interpretations](disproven.md)

## Subsystems

- [Bluetooth](bluetooth.md)
- [iAP2](iap2.md)
- [Wi-Fi](wifi.md)
- [AirPlay](airplay.md)
- [CarPlay / DIO](carplay.md)

## Reference / Methodology

- [Binary Inventory](binaries.md)
- [Firmware Provenance](firmware.md)
- [Glossary](glossary.md)
- [SSH / Runtime Environment](ssh-environment.md)
- [Function-Level Call Graphs](call-graphs/README.md)
- [Execution / Data-Flow Traces](../traces/README.md)
- [Documentation Source of Truth](source-of-truth.md)

## Retired

- `wireless-capability-breakdown.md` — retired because its broad scope duplicated the subsystem and architecture documents. Its useful findings should be maintained at their authoritative subsystem location.

The repository's authority hierarchy is defined in [Documentation Source of Truth](source-of-truth.md). Completed traces are the execution-level source; the evidence register records the accepted evidence behind conclusions; subsystem and architecture documents synthesize those lower layers.

## Evidence Discipline

Subsystem documents may contain detailed research. The evidence register is the cross-project index of what is currently accepted as proven, partially traced, unproven or disproven. The call-graph layer is reserved for actual function/process execution paths.
