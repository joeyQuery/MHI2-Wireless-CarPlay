# MHI2 Wireless CarPlay Roadmap

This roadmap is dependency-oriented. A checkbox means the repository has established the item to the stated evidence standard; it does not mean a plausible architecture exists on paper.

## 1. Evidence / Baseline

- [x] Establish Marvell 8787 WLAN/BT architecture
- [x] Establish Wi-Fi/AP infrastructure
- [x] Establish Bluetooth infrastructure
- [x] Establish iAP/iAP2 components
- [x] Establish AirPlay/mDNS infrastructure
- [x] Establish DIO as a CarPlay integration boundary
- [x] Establish the production USB CarPlay transport
- [x] Establish evidence/disproof documentation structure
- [ ] Record complete firmware provenance for every analysis baseline

## 2. Bluetooth → iAP2

- [ ] Resolve `enableIap` parsing and control flow
- [ ] Trace `bluetooth` → Bluetooth iAP proxy
- [ ] Trace Bluetooth iAP → iAP2 transport creation
- [ ] Identify the production HCI transport used by the relevant path
- [ ] Trace iAP2 callbacks/events into DIO

**Dependency:** Bluetooth/iAP2 work must establish the real transport boundary before implementation changes are selected.

## 3. DIO iAP2 Boundary

- [ ] Trace `CIpodAP2Service`
- [ ] Trace `/dev/ipod0` open/use path
- [ ] Determine whether the service is transport-neutral
- [ ] Identify the exact adaptation point for wireless iAP2
- [ ] Recover connection/disconnection state transitions

**Dependency:** Requires the Bluetooth/iAP2 trace above.

## 4. Wi-Fi / mDNS / AirPlay

- [ ] Find every consumer of `MDNS_DIRECTLINK_IFACE`
- [ ] Trace interface selection from configuration to socket setup
- [ ] Trace DIO → AirPlay interface/transport setters
- [ ] Recover `SetIFName` argument
- [ ] Recover `SetTransportType` argument
- [ ] Recover `SetClientIfMACAddr` argument
- [ ] Determine whether AirPlay can operate on `uap0` without modification
- [ ] Correlate mDNS discovery with DIO session events

**Dependency:** Interface-selection decisions should be based on the actual DIO/AirPlay call path, not on replacing `carplay0` by assumption.

## 5. Session Correlation

- [ ] Correlate Bluetooth/iAP2 phone identity with Wi-Fi association
- [ ] Correlate iAP2 events with DIO CarPlay request/state
- [ ] Correlate mDNS/AirPlay discovery with the same DIO session
- [ ] Establish one end-to-end timestamped Wireless CarPlay trace

## 6. Media / Control Validation

- [ ] Validate screen stream
- [ ] Validate audio stream
- [ ] Validate HID/control
- [ ] Validate session mode changes
- [ ] Validate disconnect/finalization
- [ ] Validate reconnection

## 7. Implementation

Implementation should begin only after the relevant transport boundaries above are proven sufficiently to identify the correct adaptation point.

- [ ] Establish Wireless CarPlay Bluetooth bootstrap
- [ ] Establish wireless iAP2 transport
- [ ] Connect wireless iAP2 to DIO
- [ ] Bind the network path to the verified interface
- [ ] Connect wireless discovery to the existing AirPlay path
- [ ] Achieve a working Wireless CarPlay session
- [ ] Validate full session lifecycle

## Dependency Graph

```text
Bluetooth
   │
   ├── enableIap
   │      ↓
   │   Bluetooth iAP
   │      ↓
   │   wireless iAP2
   │      ↓
   │   DIO iAP2 boundary
   │             │
   │             ├──────────────┐
   │             ▼              │
   │          CarPlay           │
   │             ▲              │
   │             │              │
   └─────────────┘              │
                                │
uap0 → mDNS → AirPlay ────────────┘
```

The graph shows convergence targets, not proven call edges.

## Completion Criterion

The investigation is complete when the major Wireless CarPlay path can be described as a reproducible chain of:

```text
process → library → function → IPC/transport → device/socket → protocol event → session state
```

with each important edge backed by an evidence-register entry.
