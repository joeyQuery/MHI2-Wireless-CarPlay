# Repository Status

> **Project:** MHI2 Wireless CarPlay  
> **Platform:** Audi MHI2 / MU0678-class QNX  
> **Purpose:** Evidence-driven reverse engineering and implementation of Wireless Apple CarPlay using the existing MHI2 connectivity and CarPlay stack.

## Current State

The repository has mapped the major MHI2 subsystems relevant to Wireless CarPlay:

- Marvell 8787 WLAN/BT hardware
- WLAN/AP infrastructure and `uap0`
- Bluetooth and `btstack`
- iAP/iAP2 infrastructure
- USB CarPlay networking via `carplay0`
- DIO CarPlay integration
- AirPlay receiver and Bonjour/mDNS infrastructure
- existing Bluetooth/WLAN coexistence

The production USB architecture is substantially mapped. The Wireless CarPlay architecture is **not yet proven end-to-end**.

## Current Integration Boundary

```text
PRODUCTION

iPhone
 ├─ USB → /dev/ipod0 → iAP2 → DIO
 └─ USB → devnp-usbdnet.so → carplay0 → DIO / AirPlay

TARGET

iPhone
 ├─ Bluetooth → wireless iAP2 → DIO
 └─ Wi-Fi → uap0 → mDNS / AirPlay → DIO
```

The target diagram describes the investigation target, not a completed implementation.

## Highest-Value Open Questions

1. What exact code branch in `bluetooth` consumes `enableIap=false`?
2. What exact endpoint path does `IapDeviceServices` return to `CIapBTChannel`?
3. Does that Bluetooth endpoint equal DIO's `/dev/ipod0`, or is it a different resource-manager path?
4. How does the production boot path export `MDNS_DIRECTLINK_IFACE=carplay0` into the running `mdnsd` environment?
5. Where is the AirPlay object's `interfaceName` populated, and what runtime value does it hold?
6. What callers/arguments drive the substantive AirPlay packet/multicast interface helpers?
7. How are Bluetooth/iAP2 and Wi-Fi/AirPlay associated with the same phone/session?
8. Can the recovered AirPlay/mDNS path operate on `uap0` without modification?
9. What production activation branch connects the existing Bluetooth/Wi-Fi iAP2 machinery to the DIO CarPlay state machine?
10. What concrete transport/device object is selected after the Bluetooth `CIapBTChannel` runtime endpoint reaches `open64()`?
11. What callers reach `libiap2client.so::iap2_connect()`, and what QNX message/resource-manager destination do those calls address?
12. Does the transport object/callback layer in `ipod-drvr-iap2.so` own or mediate either of those two boundaries?
13. How, if at all, can that Bluetooth-side iAP2 transport reach DIO when production DIO is configured for `/dev/ipod0`?

## Evidence Rules

Every new finding should be classified as one or more of:

- **Static evidence** — strings, symbols, configuration, disassembly.
- **Runtime evidence** — observed process, interface, session or system behaviour.
- **Protocol evidence** — HCI, iAP2, mDNS or AirPlay traces.
- **Controlled experiment** — deliberate modification followed by observation.
- **Inference** — interpretation that still requires MHI2-specific confirmation.

See [Evidence](docs/evidence.md) for the authoritative evidence register and [Disproven](docs/disproven.md) for eliminated interpretations.

## Current Documentation Layers

- [Wireless CarPlay Architecture](docs/wireless-carplay-architecture.md) — end-to-end integration map.
- [Bluetooth](docs/bluetooth.md) — Bluetooth subsystem.
- [iAP2](docs/iap2.md) — iAP/iAP2 subsystem and transport boundary.
- [Wi-Fi](docs/wifi.md) — WLAN/AP subsystem.
- [AirPlay](docs/airplay.md) — AirPlay, Bonjour and interface selection.
- [CarPlay](docs/carplay.md) — DIO/CarPlay integration.
- [Binaries](docs/binaries.md) — binary/process inventory.
- [Firmware](docs/firmware.md) — firmware provenance and analysis baseline.
- [Glossary](docs/glossary.md) — project terminology.
- [Evidence](docs/evidence.md) — evidence register.
- [Disproven](docs/disproven.md) — eliminated interpretations.
- [SSH Environment](docs/ssh-environment.md) — runtime investigation environment.
- [Call Graphs](docs/call-graphs/README.md) — function-level tracing structure.
- [Execution Traces](traces/README.md) — transport-boundary trace artifacts.
- [Source of Truth](docs/source-of-truth.md) — documentation authority hierarchy.

## Explicitly Retired

`docs/wireless-capability-breakdown.md` was a broad aggregation that overlapped the subsystem documents. Its unique useful material should live in the appropriate subsystem/connectivity documentation rather than maintaining a second architecture source.

## Trace State

The repository now has ten explicit trace artifacts for the highest-value transport boundaries. TRACE-001, TRACE-003, TRACE-004 and TRACE-005 have gained concrete binary-level edges; TRACE-002 remains partial because the DIO transport adaptation point is unresolved; TRACE-006 remains a target because end-to-end identity correlation has not been demonstrated; TRACE-007 and TRACE-008 are partial, documenting compiled multi-transport iAP2 capability and the recovered iAP2 control-plane/DIO Bluetooth integration boundaries without proving production activation. TRACE-010 now isolates the iAP2 client/media-synchronizer boundary and records `ipod-drvr-iap2.so` as the strongest architectural candidate for a common transport layer, while still leaving the Bluetooth-to-driver/DIO convergence edge unresolved. These documents must not be read as proof of the remaining runtime/session edges.

## Immediate Research State

The project is currently in the **transport-boundary tracing** phase. The next implementation decisions should be based on concrete traces of Bluetooth/iAP2, DIO, mDNS and AirPlay rather than on the existence of firmware components alone.
