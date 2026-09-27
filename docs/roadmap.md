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
- [x] Trace Bluetooth active-device → iAP endpoint creation boundary
- [ ] Resolve the exact runtime endpoint path returned by `IapDeviceServices`
- [ ] Trace Bluetooth iAP → iAP2 transport creation
- [ ] Identify the production HCI transport used by the relevant path
- [ ] Trace iAP2 callbacks/events into DIO

**Dependency:** Bluetooth/iAP2 work must establish the real transport boundary before implementation changes are selected.


## 2.1 MU0678 iAP2 transport-capability finding

The MU0678 ipod-drvr-iap2.so binary contains generic transport callbacks plus compiled Bluetooth and Wi-Fi transport identification. The Wi-Fi transport descriptor explicitly carries iAP2-connection and CarPlay capability fields.

The shipped /etc/mm/iap2.cfg nevertheless selects Lightning Connector and leaves the Bluetooth section commented out.

The investigation boundary is therefore narrowed: the driver demonstrably has a wireless transport model, but the production wireless transport object/callback implementation, selection mechanism, and DIO handoff are still unresolved.

## 3. DIO iAP2 Boundary

- [ ] Trace `CIpodAP2Service`
- [ ] Trace `/dev/ipod0` open/use path
- [ ] Determine whether the service is transport-neutral
- [ ] Identify the exact adaptation point for wireless iAP2
- [ ] Recover connection/disconnection state transitions

**Dependency:** Requires the Bluetooth/iAP2 trace above.

## 4. Wi-Fi / mDNS / AirPlay

- [x] Identify the `mdnsd` consumer of `MDNS_DIRECTLINK_IFACE`
- [ ] Prove how the production boot path exports `MDNS_DIRECTLINK_IFACE=carplay0` into the running `mdnsd` environment
- [x] Trace `MDNS_DIRECTLINK_IFACE` through `mdnsd::SetupOneInterface()`
- [ ] Trace final interface selection from mDNS/AirPlay into socket setup
- [x] Determine that the production screen interface/transport/client-MAC setters are no-op stubs
- [ ] Trace the actual AirPlay interface-selection path
- [x] Retire the three screen setters as the presumed active transport selector
- [ ] Recover where the AirPlay object's `interfaceName` field is populated
- [ ] Recover packet/multicast interface helper arguments
- [ ] Determine whether the recovered `interfaceName`/socket path can operate on `uap0` without modification
- [ ] Correlate mDNS discovery with DIO session events

**Dependency:** Interface-selection decisions should be based on the actual DIO/AirPlay call path, not on replacing `carplay0` by assumption.

## 4.1 Newly recovered boundary state

The highest-value transport unknowns have narrowed substantially. Bluetooth iAP now has a proven runtime endpoint handoff into `open64()`, while DIO independently proves `/dev/ipod0`; the endpoint equality is unresolved. AirPlay now has a proven Bonjour interface-index path and a real `libairplay` → DNS-SD → `mdnsd` boundary. The remaining mDNS unknown is environment provenance, not consumer identity.

## 5. Session Correlation

- [ ] Correlate Bluetooth/iAP2 phone identity with Wi-Fi association and the runtime Bluetooth endpoint
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



## 2.2 MU0678 iAP2 control-plane finding

- [x] Establish executable Bluetooth iAP2 feature-startup machinery
- [x] Establish shared iAP2 packet-dispatch handling for Bluetooth and Wi-Fi control events
- [x] Establish accessory Wi-Fi configuration response fields
- [x] Establish DIO Bluetooth Smartphone Integration / CarPlay control boundary
- [ ] Prove production activation of these wireless paths
- [ ] Correlate the selected iAP2 transport object with DIO

The new trace narrows the problem from capability discovery to production activation and handoff. These checkboxes do not imply a working Wireless CarPlay session. See TRACE-008 and E-039 through E-043.
