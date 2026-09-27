# MHI2 Wireless CarPlay Traces

This directory contains execution/data-flow traces. It is deliberately separate from subsystem descriptions.

## Trace status

A trace may be:

- **Complete** — the important execution/data edges are recovered and evidence-backed.
- **Partial** — some edges are established, but one or more boundaries remain unresolved.
- **Target** — an investigation plan only; no recovered execution edge is claimed.

No target arrow is evidence by itself.

## Trace index

| Trace | Scope | Status |
|---|---|---|
| [TRACE-001](TRACE-001-bluetooth-iap2.md) | Bluetooth → iAP2 | Partial |
| [TRACE-002](TRACE-002-dio-iap2.md) | DIO → iAP2 transport | Partial |
| [TRACE-003](TRACE-003-mdns-interface.md) | MDNS_DIRECTLINK_IFACE | Partial |
| [TRACE-004](TRACE-004-dio-airplay.md) | DIO → AirPlay | Partial |
| [TRACE-005](TRACE-005-airplay-network.md) | AirPlay → socket/interface | Partial |
| [TRACE-006](TRACE-006-session-correlation.md) | Bluetooth/Wi-Fi/session correlation | Target |
| [TRACE-007](TRACE-007-iap2-multitransport.md) | MU0678 iAP2 multi-transport capability | Partial |\n| [TRACE-008](TRACE-008-iap2-control-plane.md) | MU0678 iAP2 control plane / DIO Bluetooth integration | Partial |

## Trace record format

Every trace should record the following fields, even when the value is `Unresolved`, `Not captured`, or `Not applicable`:

- entry point;
- process/binary;
- function/symbol and address when known;
- caller/callee;
- arguments and return/error behaviour when recovered;
- IPC/ASI/DSI boundary;
- device/socket/file boundary;
- protocol event;
- runtime confirmation;
- evidence IDs;
- remaining uncertainty.

## Source-of-truth rule

Completed trace facts outrank architectural prose. See [Source of Truth](../docs/source-of-truth.md).

Raw runtime captures and experiment repositories are intentionally outside this directory. Therefore a trace may cite a runtime observation without being repository-reproducible; that limitation must be stated explicitly in the trace record.
