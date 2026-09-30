# MHI2 Wireless CarPlay Roadmap

This roadmap is dependency-oriented. A checkbox means the repository has established the item to the stated evidence standard; it does not mean that Wireless CarPlay is implemented.

## 1. Production baseline

- [x] Establish Marvell 8787 shared WLAN/BT architecture
- [x] Establish WLAN/AP infrastructure and `uap0`
- [x] Establish Bluetooth infrastructure
- [x] Establish production Bluetooth iAP/iAP2 implementation
- [x] Establish DIO as the CarPlay integration boundary
- [x] Establish the production USB iAP2 path through `/dev/ipod0`
- [x] Establish the production USB CarPlay network path through `carplay0`
- [x] Establish AirPlay / Bonjour / mDNS infrastructure
- [x] Establish evidence and trace hierarchy
- [ ] Complete firmware provenance for every analysis baseline

## 2. Bluetooth iAP2 endpoint

- [x] Identify `CIapBTChannel` and its open/connect/read/write/close path
- [x] Identify the `CIapConnectorMachine` state-machine boundary
- [x] Identify the `IapDeviceServices` RPC boundary
- [x] Prove active-device information reaches `CIapBTChannel::updateiAPDevice()`
- [x] Prove the Bluetooth endpoint reaches `open64()`
- [x] Establish that the Bluetooth `iap` binary does not hard-code `/dev/ipod0`
- [ ] Resolve the `enableIap` configuration parser/control-flow branch
- [ ] Recover the exact endpoint value/path supplied by the Bluetooth service
- [ ] Identify the owner/creator of that endpoint
- [ ] Determine whether the endpoint is the same mounted QNX iAP2 resource-manager service used by `ipod-drvr-iap2.so`
- [ ] Correlate the Bluetooth endpoint with the concrete iAP2 message ABI proven in TRACE-012
- [ ] Trace the resulting iAP2 session into DIO

**Current boundary:** TRACE-013. The Bluetooth implementation is proven; the endpoint ownership and DIO handoff are not.

## 3. DIO iAP2 boundary

- [x] Trace DIO `CIpodAP2Service`
- [x] Prove DIO passes a caller-supplied path into `iap2_connect()`
- [x] Prove `/dev/ipod0` is the current production configuration rather than an intrinsic `iap2_connect()` restriction
- [x] Prove the iAP2 client/driver resource-manager ABI
- [ ] Determine whether DIO can consume the Bluetooth-side endpoint without a new transport ABI
- [ ] Recover the iAP2 connection/disconnection state transitions for the wireless path
- [ ] Establish the exact Bluetooth iAP2 → DIO handoff

**Dependency:** the Bluetooth endpoint trace must identify the actual endpoint before an implementation point is selected.

## 4. Wi-Fi / mDNS / AirPlay

- [x] Establish Marvell AP infrastructure and `uap0`
- [x] Establish DHCP/DNS/PF support around `uap0`
- [x] Establish production `MDNS_DIRECTLINK_IFACE=carplay0`
- [x] Prove `mdnsd` consumes `MDNS_DIRECTLINK_IFACE`
- [x] Prove AirPlay Bonjour registration uses its `interfaceName` field
- [x] Prove the production screen interface/transport/client-MAC setters are no-op stubs
- [x] Prove substantive packet-receive and multicast interface helpers exist
- [ ] Recover where AirPlay `interfaceName` is populated
- [ ] Trace callers and arguments of `SocketSetPacketReceiveInterface`
- [ ] Trace callers and arguments of `SocketSetMulticastInterface`
- [ ] Prove how the production boot path exports `MDNS_DIRECTLINK_IFACE=carplay0` into `mdnsd`
- [ ] Determine whether the recovered AirPlay/mDNS path can operate on `uap0`
- [ ] Correlate mDNS/AirPlay discovery with DIO session events

**Important correction:** do not use `AirPlayReceiverSessionScreen_SetIFName()`, `SetTransportType()`, or `SetClientIfMACAddr()` as evidence of active interface selection on MU0678. They are production no-op stubs.

## 5. Bluetooth ↔ Wi-Fi session correlation

- [ ] Correlate Bluetooth phone identity with Wi-Fi association
- [ ] Correlate Bluetooth/iAP2 endpoint identity with the same phone
- [ ] Correlate iAP2 events with DIO CarPlay request/state
- [ ] Correlate mDNS/AirPlay discovery with the same DIO session
- [ ] Produce one timestamped end-to-end Wireless CarPlay trace

## 6. Session/media validation

Only begin this section after the transport boundaries above are sufficiently proven.

- [ ] Validate CarPlay session creation
- [ ] Validate video
- [ ] Validate audio
- [ ] Validate HID/control
- [ ] Validate mode changes
- [ ] Validate disconnect/finalization
- [ ] Validate reconnection

## 7. Implementation

Implementation should follow the recovered production architecture rather than introducing a parallel CarPlay stack.

- [ ] Establish the production Bluetooth bootstrap path
- [ ] Establish the actual wireless iAP2 transport endpoint
- [ ] Connect that endpoint to the existing DIO iAP2 boundary
- [ ] Bind the verified Wi-Fi/mDNS/AirPlay path to the correct interface
- [ ] Establish Bluetooth/Wi-Fi session correlation
- [ ] Achieve a working Wireless CarPlay session
- [ ] Validate the complete lifecycle

## Dependency graph

```text
Bluetooth
   |
   v
IapDeviceServices
   |
   v
CIapBTChannel
   |
   v
runtime endpoint
   |
   +---- [unresolved endpoint owner / ABI]
   |
   v
iAP2 resource-manager ?
   |
   v
DIO / CarPlay
   |
   +----------------------+
   |                      |
   v                      v
iAP2                  AirPlay / Bonjour
                          |
                          v
                   mDNS / socket layer
                          |
                          v
                         uap0 ?
```

Only the solid production components and recovered edges should be treated as evidence. The `?` and unresolved branches are investigation targets.

## Current completion criterion

The investigation is complete when the major Wireless CarPlay path can be described as a reproducible chain of:

```text
process → library → function → IPC/transport → device/socket
→ protocol event → session state
```

with each important edge backed by an evidence-register entry.

Wireless CarPlay should not be marked implemented or end-to-end proven until the Bluetooth endpoint/DIO handoff and the Wi-Fi/AirPlay interface-selection path are supported by MHI2-specific evidence.

**Latest trace:** [TRACE-013 — Bluetooth iAP2 → DIO / AirPlay Boundary](../traces/TRACE-013-bluetooth-iap2-dio-airplay-boundary.md)
