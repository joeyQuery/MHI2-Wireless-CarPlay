# TRACE-013 — MU0678 Bluetooth iAP2 → DIO / AirPlay Boundary

**Status:** Partial

## Objective

Consolidate the latest production-binary trace of the Bluetooth iAP2 path and identify the remaining binary-level boundary before Wireless CarPlay can be considered end-to-end proven.

## Firmware baseline

- Platform: Audi MHI2 / MU0678-class QNX
- Production image: MU0678 application image used throughout the repository
- Relevant binaries:
  - `eso/bin/apps/iap`
  - `eso/bin/apps/dio_manager`
  - `eso/lib/factories/libasimmxconnectivity_bluetooth_iapproxy.so`
  - `eso/lib/libairplay.so`
  - `eso/bin/apps/mdnsd`

## 1. Bluetooth iAP2 is a production subsystem

The production `iap` binary contains a dedicated Bluetooth transport implementation:

```text
iap::CIapBTChannel
  openiAPDevice()
  connectToiAPDevice()
  readiAPDevice()
  writeiAPDevice()
  updateiAPDevice()
  closeiAPDevice()
```

It also contains `CIapConnectorMachine`, including iAP support checks and state transitions, plus iAP2 SYN/SYNACK/ACK/link-packet machinery.

The Bluetooth path is therefore more than pairing or dormant capability.

**Evidence:** E-014, E-015, E-021, E-022.

## 2. Bluetooth iAP uses an RPC/service boundary

The production image contains:

```text
asi.connectivity.bluetooth.iap.IapDeviceServices
```

in `libasimmxconnectivity_bluetooth_iapproxy.so`, with matching service/reply machinery in `iap`.

The recovered architecture is:

```text
Bluetooth connectivity service
        |
        | COMM/RPC
        v
IapDeviceServices
        |
        v
iap / CIapBTChannel
        |
        +--> open64()
        +--> read()
        +--> write()
        +--> close()
```

The exact endpoint publication/consumer path remains unresolved.

**Evidence:** E-015, E-021, E-022.

## 3. The Bluetooth endpoint is not proven to be /dev/ipod0

`CIapBTChannel::openiAPDevice()` reaches `open64()` with a runtime-supplied endpoint. No `/dev/ipod0` literal was recovered from the `iap` binary.

Separately, DIO's production USB iAP2 path is:

```text
dio_manager
  -> CIpodAP2Service
  -> iap2_connect()
  -> configured /dev/ipod0
```

This proves a transport/path separation. It does **not** prove that the Bluetooth endpoint is incompatible with the iAP2 resource-manager ABI.

**Evidence:** E-016, E-022, E-023, E-049, E-050, E-051, E-054, E-055.

## 4. DIO/AirPlay network boundary

DIO contains production AirPlay lifecycle integration and the configuration value:

```text
MDNS_DIRECTLINK_IFACE=carplay0
```

The value is part of DIO configuration. The existing `carplay0` path is the USB CarPlay network interface.

The production `libairplay.so` also contains real:

```text
SocketSetPacketReceiveInterface
SocketSetMulticastInterface
IsWiFiNetworkInterface
DNSService*
```

network machinery.

However, the following exported screen setters are production no-op stubs and must not be treated as the active wireless-interface selector:

```text
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
```

**Evidence:** E-017, E-018, E-019, E-020, E-025, E-026, E-027, E-028, E-029, E-030.

## 5. Current recovered architecture

```text
Bluetooth
   |
   v
IapDeviceServices COMM/RPC
   |
   v
iap / CIapBTChannel
   |
   v
runtime-supplied endpoint
   |
   +---- [unresolved bluetooth-side handoff]
   |
  DIO
   |
   +---- iAP2 / CarPlay state
   |
   +---- AirPlay / Bonjour
              |
              v
        mDNS / socket layer
              |
              v
          Wi-Fi path
```

This is an evidence-oriented convergence diagram. The dashed/unresolved handoff is not a proven execution edge.

## 6. Remaining blocker

The remaining static question is now specific:

> What exact endpoint does the Bluetooth service supply to `CIapBTChannel::openiAPDevice()`, who owns that endpoint, and does it terminate at the same QNX iAP2 resource-manager ABI implemented by `ipod-drvr-iap2.so)?

The production `bluetooth` executable is the highest-value missing artifact for resolving the first half of this question. Without it, the endpoint construction/ownership branch must remain unresolved.

A runtime capture of the endpoint would also resolve the ambiguity if the binary remains unavailable.

## 7. AirPlay follow-up

The network-side investigation should continue independently of the Bluetooth endpoint:

1. Recover where the AirPlay object's `interfaceName` is populated.
2. Trace callers/arguments to `SocketSetPacketReceiveInterface`.
3. Trace callers/arguments to `SocketSetMulticastInterface`.
4. Establish whether the resulting path can operate on `uap0` without modification.
5. Keep `MDNS_DIRECTLINK_IFACE=carplay0` classified as the current production configuration, not as proof that `uap0` is already selected.

## Evidence summary

- **Proven:** production Bluetooth iAP2 implementation and RPC boundary.
- **Proven:** Bluetooth endpoint is runtime-supplied to `open64()`.
- **Proven:** DIO's current configured iAP2 path is `/dev/ipod0`.
- **Proven:** AirPlay has substantive Wi-Fi/interface/socket primitives.
- **Disproven/corrected:** the three screen interface/transport setters are not active selectors in this production build.
- **Partial:** Bluetooth endpoint → iAP2 resource-manager/DIO handoff.
- **Partial:** DIO/AirPlay runtime interface selection.
- **Not yet proven:** end-to-end Wireless CarPlay.

## Related evidence

E-014 through E-031, E-049 through E-055.

## Related traces

- TRACE-001 — Bluetooth → iAP2
- TRACE-002 — DIO → iAP2 transport
- TRACE-003 — MDNS_DIRECTLINK_IFACE
- TRACE-004 — DIO → AirPlay
- TRACE-005 — AirPlay → socket/interface
- TRACE-008 — iAP2 control plane / DIO Bluetooth integration
- TRACE-011 — DIO runtime iAP2 endpoint
- TRACE-012 — iAP2 client → driver resource-manager boundary
