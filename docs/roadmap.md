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
- [x] Resolve the `enableIap` configuration parser/control-flow branch
- [ ] Recover the exact endpoint value/path supplied by the Bluetooth service
- [ ] Identify the owner/creator of that endpoint
- [ ] Determine whether the endpoint is the same mounted QNX iAP2 resource-manager service used by `ipod-drvr-iap2.so`
- [ ] Correlate the Bluetooth endpoint with the concrete iAP2 message ABI proven in TRACE-012
- [ ] Determine whether Bluetooth iAP2 is a bootstrap/control-plane path separate from DIO's session-side iAP2
- [ ] Trace the resulting Bluetooth bootstrap into Wi-Fi session activation
- [ ] Determine whether MU0678 has a session-side iAP2-over-AirPlay path comparable to the MH2p reference

**Current boundary:** TRACE-013. The Bluetooth implementation is proven; the endpoint ownership and DIO handoff are not.

## 3. DIO iAP2 boundary

- [x] Trace DIO `CIpodAP2Service`
- [x] Prove DIO passes a caller-supplied path into `iap2_connect()`
- [x] Prove `/dev/ipod0` is the current production configuration rather than an intrinsic `iap2_connect()` restriction
- [x] Prove the iAP2 client/driver resource-manager ABI
- [ ] Determine whether DIO can consume the Bluetooth-side endpoint without a new transport ABI
- [ ] Recover the iAP2 connection/disconnection state transitions for the wireless path
- [ ] Establish whether there is any direct Bluetooth iAP2 → DIO handoff (do not assume one)
- [ ] Establish the actual wireless-session handoff into DIO

**Dependency:** the Bluetooth endpoint trace must identify the actual endpoint before an implementation point is selected.

## 4. Wi-Fi / mDNS / AirPlay

- [x] Establish Marvell AP infrastructure and `uap0`
- [x] Establish DHCP/DNS/PF support around `uap0`
- [x] Establish production `MDNS_DIRECTLINK_IFACE=carplay0`
- [x] Prove `mdnsd` consumes `MDNS_DIRECTLINK_IFACE`
- [x] Prove AirPlay Bonjour registration uses its `interfaceName` field
- [x] Prove the production screen interface/transport/client-MAC setters are no-op stubs
- [x] Prove substantive packet-receive and multicast interface helpers exist
- [ ] Recover where AirPlay `interfaceName` is populated in MU0678
- [x] Close the GLOB_DAT linkage question for `AirPlayReceiverServerSetProperty` (active indirect callback proven)
- [ ] Identify the property object/name supplied at the recovered MU0678 callback sites
- [ ] Trace callers and arguments of `SocketSetPacketReceiveInterface`
- [ ] Trace callers and arguments of `SocketSetMulticastInterface`
- [ ] Prove how the production boot path exports `MDNS_DIRECTLINK_IFACE=carplay0` into `mdnsd`
- [ ] Determine whether the recovered AirPlay/mDNS path can operate on `uap0`
- [ ] Correlate mDNS/AirPlay discovery with DIO session events

**Important correction:** do not use `AirPlayReceiverSessionScreen_SetIFName()`, `SetTransportType()`, or `SetClientIfMACAddr()` as evidence of active interface selection on MU0678. They are production no-op stubs.

## 5. Bluetooth ↔ Wi-Fi session correlation

- [ ] Correlate Bluetooth phone identity with Wi-Fi association
- [ ] Correlate Bluetooth/iAP2 endpoint identity with the same phone
- [ ] Correlate Bluetooth bootstrap/iAP2 events with DIO CarPlay request/state
- [ ] Determine whether session-side iAP2 is carried through AirPlay on MU0678
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
Bluetooth bootstrap
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
   +---- [MU0678 endpoint owner unresolved]
   |
   v
bootstrap iAP2 / phone identity ?
   |
   v
Wi-Fi AP / uap0
   |
   v
AirPlay / Bonjour
   |
   v
DIO / CarPlay
   |
   +---- session-side iAP2 over AirPlay ?

```

Only the solid production components and recovered edges should be treated as evidence. The `?` and unresolved branches are investigation targets.

## Current completion criterion

The investigation is complete when the major Wireless CarPlay path can be described as a reproducible chain of:

```text
process → library → function → IPC/transport → device/socket
→ protocol event → session state
```

with each important edge backed by an evidence-register entry.

Wireless CarPlay should not be marked implemented or end-to-end proven until the MHI2-specific Bluetooth bootstrap path, wireless-session handoff into DIO, session-side iAP2/AirPlay capability, and Wi-Fi/AirPlay interface-selection path are supported by MHI2-specific evidence. Do not require a direct Bluetooth endpoint → DIO `/dev/ipod0` handoff unless MHI2 evidence actually establishes that architecture.

**Latest trace:** [TRACE-013 — Bluetooth iAP2 → DIO / AirPlay Boundary](../traces/TRACE-013-bluetooth-iap2-dio-airplay-boundary.md)


### 2026-09-30 transport-boundary update

The MU0678 bluetooth ELF is now directly inspected. Its enableIap/iapEnabled policy strings and switchBluetoothAccordingToConfig() branch are proven, but the exact config-field mapping is not yet closed. The high-level bluetooth executable also does not expose a recovered concrete iAP endpoint-construction/RFCOMM-server API.

The MH2p reference provides a concrete reference-only endpoint pattern: btstack creates /dev/iapDevice-<BT address> after the iAP2 probe bytes and reports it through updIapDevicePath. This is now the exact byte/string/function signature set to test against MU0678. It must not be promoted to MU0678 evidence until the MU0678 btstack ELF or a runtime capture confirms it.
### 2026-09-30 MU0678 btstack endpoint closure

The exact MU0678 btstack ELF is now directly inspected. The previous endpoint-owner blocker is closed: btstack contains btstack::IapDevice, a QNX resmgr_attach path, the compiled /dev/iapDevice endpoint base, IapServices register/deregister logic, and an iAP2 accessory SDP record that explicitly contains RFCOMM plus UUID 00000000-deca-fade-deca-deafdecacaff. The SDP record is directly adjacent to the IapDeviceServices registration name and the IapServices registration path directly references the record.

The remaining static questions are now implementation details: the runtime suffix after /dev/iapDevice, the exact writer of the SDP RFCOMM channel byte, the lower-level local/indirect BlueSDK RFCOMM callback, and the exact connected-BT-address -> endpoint publication edge. The MH2p /dev/iapDevice-<BT-address> naming convention remains reference-only until MU0678-specific evidence closes it.

Do not describe bluetooth.enableIap as the btstack startup or endpoint-creation switch. Its direct MU0678 consumer is still the Bluetooth topology/reconnect policy path traced in TRACE-013.


### 2026-09-30 RFCOMM channel / endpoint-path refinement

The MU0678 btstack trace now closes the SDP-byte writer. IapServices registration helper `0x23799c` reads `IapServices + 0x0c` and writes that value into the RFCOMM channel byte at `0x2d2519` of the iAP2 SDP record at `0x2d250c`, then calls lower-level SDP routine `0x20be48`. The producer of `IapServices + 0x0c` is still unresolved; it must not be confused with the SDP record or promoted to a named BlueSDK API without proof.

The endpoint path is also narrowed: `0x23c720` loads the exact `/dev/iapDevice` string, constructs an iPL string-state object, and passes it into the resource-manager construction path ending at `resmgr_attach()` `0x23fc84`. MU0678-specific evidence still does not prove the MH2p `/dev/iapDevice-<BT address>` suffix convention.

The remaining static endpoint targets are therefore the producer of the IapServices channel byte and the BT-address-to-final-endpoint pathname data-flow. Everything upstream of those boundaries is now concrete MU0678 evidence.

### 2026-09-30 IapServices embedded-event boundary

The remaining RFCOMM-channel source was traced into shared btstack machinery. IapServices passes `this+0x10` and `this+0x0c` to shared routine `0x242574`; that routine consumes byte 0 of the `+0x0c` structure and calls lower-level routine `0x2081ac`. Other callers prove `0x242574` is shared machinery. No direct IapServices byte-0 writer was recovered, so the exact event producer/indirect BlueSDK callback remains unresolved.

This is the current static exhaustion boundary for the channel source. Do not invent a BlueSDK function name or map the byte to a BT address without runtime or additional binary evidence.

### 2026-09-30 exhaustive twelve-target static campaign

TRACE-014 re-audits all twelve remaining static targets in one pass: iAP2 transport construction, Wi-Fi configuration/AP handoff, DIO iAP2/session transition, IapDeviceServices publication, btstack endpoint naming, IapServices RFCOMM channel production, DIO AirPlay interfaceName population, AirPlay socket-helper callers, MDNS_DIRECTLINK_IFACE propagation, BluetoothSmartphoneIntegration/CarPlay state, ipod-drvr-iap2 mountpoint construction, and boot/configuration dependency ordering.

The important result is not that Wireless CarPlay is complete; it is that each target now has a concrete stopping boundary. The MU0678 btstack endpoint is proven through btstack::IapDevice, /dev/iapDevice, iPL string-state construction and resmgr_attach, while a direct search proves that the MH2p /dev/iapDevice-<BT-address> and updIapDevicePath literals are absent from the exact MU0678 btstack ELF. The RFCOMM channel source is narrowed to IapServices +0x0c → shared 0x242574 → 0x2081ac, with no direct byte-0 writer recovered.

The remaining open items are therefore the exact producer/consumer edges listed in TRACE-014, not a need to rediscover the surrounding subsystems.

**Latest trace:** [TRACE-014 — Exhaustive Static Campaign: 12 Remaining Wireless CarPlay Traces](../traces/TRACE-014-exhaustive-static-campaign-12.md)


### 2026-09-30 expanded static mapping update — TRACE-015

Two previously unresolved boundaries are now closed.

**Bluetooth iAP policy:** `bluetooth.enableIap` is proven to be the property read into `CBluetoothTopologyReconnect +0x28` by the constructor at `0x155178` through the configuration reader `0x155060 -> 0x154c3c`. The cached byte is consumed by `0x17b280` as Bluetooth topology/reconnect policy. This does not make it a btstack startup switch.

**RFCOMM channel:** the IapServices channel byte is no longer an unresolved indirect source. `0x2379f0 -> 0x242574 -> 0x2081ac` reaches the internal btstack channel allocator. When the supplied byte is zero, `0x2081ac` scans the internal channel table and writes an allocated RFCOMM channel into the IapServices `+0x0c` storage. `0x23799c` then copies that byte into SDP byte `0x2d2519` and registers the record through `0x20be48`.

This removes the RFCOMM channel producer from the remaining blocker list. The endpoint pathname and IapDeviceServices publication are still separate unresolved boundaries.

**Latest trace:** [TRACE-015 — Expanded Static Mappings](../traces/TRACE-015-expanded-static-mappings.md)
