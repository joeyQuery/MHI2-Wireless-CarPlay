# TRACE-001 — Bluetooth → iAP2

**Status:** Partial

## Objective

Recover the production Bluetooth → iAP2 bootstrap path used for CarPlay-capable device establishment.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | Bluetooth iAP active-device callback → `CIapBTChannel::updateiAPDevice()` |
| Process / binary | `bluetooth`; `btstack`; `libasimmxconnectivity_bluetooth_iapproxy.so`; iAP2 components |
| Caller / callee | `IapDeviceServices` active-device callback → `CIapDeviceServicesReplyImpl::updateActiveDevices()` → `CIapBTChannel::updateiAPDevice()` → `CIapBTChannel::openiAPDevice()` → `open64()` |
| Arguments | Runtime device object supplies a `CIString` device-path value; exact returned string is not yet recovered |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Unresolved |
| Device / socket / file boundary | Runtime Bluetooth iAP endpoint path is passed to `open64()`; exact path value unresolved |
| Protocol event | Bluetooth/iAP2 protocol transition not yet captured as one production trace |
| Runtime confirmation | Component/runtime observations exist, but no complete timestamped Bluetooth→iAP2 execution trace is committed |
| Evidence IDs | E-006, E-007, E-014, E-015, E-021, E-022, E-023 |
| Remaining uncertainty | `enableIap` control flow, exact runtime endpoint value, Bluetooth transport handoff into DIO and whether the endpoint is `/dev/ipod0` |

Recover the production path from MHI2 Bluetooth handling into iAP/iAP2 for the CarPlay bootstrap.

## Established

The firmware contains:

- `bluetooth`;
- `btstack`;
- `libasimmxconnectivity_bluetooth_iapproxy.so`;
- `/eso/bin/apps/iap`;
- `libiap2client.so`;
- related iAP2 components.

Production Bluetooth configuration contains:

```text
enableIap=false
```

Bluetooth also exposes CarPlay-related integration types and dedicated iAP proxy infrastructure.

## Newly recovered binary trace

The production `eso/bin/apps/iap` binary contains a dedicated `iap::CIapBTChannel` implementation with:

```text
openiAPDevice()       0x109a9c
connectToiAPDevice()  0x109570
closeiAPDevice()     0x108d6c
readiAPDevice()      0x108c14
writeiAPDevice()     0x108bc8
updateiAPDevice()    0x10b434
```

`CIapBTChannel::openiAPDevice()` reaches `open64()`. No `/dev/ipod0` literal was found in `iap`, so the Bluetooth channel does not reproduce DIO's USB device path inside this binary.

The same binary contains `CIapConnectorMachine` state transitions and explicit iAP2 support checks. It also contains `asi::connectivity::bluetooth::iap::IapDeviceServicesProxy`, `IapDeviceServicesReply`, and `IapDeviceServicesServiceReplyRegistration`.

The newly recovered callback path is stronger than the earlier component-only evidence: `CIapDeviceServicesReplyImpl::updateActiveDevices()` extracts an active Bluetooth device object and eventually calls `CIapBTChannel::updateiAPDevice(...)`. The supplied `CIString` is retained by the channel, and `CIapBTChannel::openiAPDevice()` later converts that stored value into a native path and reaches `open64()`. This proves that the Bluetooth iAP channel receives its endpoint at runtime rather than hard-coding a known device path in `iap`.

The exact returned endpoint string remains unresolved. In particular, `iap` contains no `/dev/ipod0` literal, while DIO independently uses `/dev/ipod0`; those two facts must not be collapsed into one endpoint claim.

The companion `libasimmxconnectivity_bluetooth_iapproxy.so` identifies `asi.connectivity.bluetooth.iap.IapDeviceServices`, `PROXY_asi_connectivity_bluetooth_iap_IapDeviceServices`, `STUB_asi_connectivity_bluetooth_iap_IapDeviceServices`, `IapDeviceServicesRPCStub`, and `IapDeviceServicesRPCStubReply`.

This establishes a concrete Bluetooth-iAP IPC/service boundary. It does not yet establish the owning Bluetooth process, endpoint publication, or the handoff from this service into DIO.

## Current trace

```text
Bluetooth connectivity service
        |
        | RPC
        v
IapDeviceServices
        |
        | active-device callback
        v
CIapDeviceServicesReplyImpl::updateActiveDevices()
        |
        v
CIapBTChannel::updateiAPDevice(...)
        |
        | runtime CIString/device path
        v
CIapBTChannel::openiAPDevice()
        |
        v
open64(path)
        |
        v
Bluetooth iAP2 device endpoint
        |
        v
readiAPDevice / writeiAPDevice
        |
        v
CIapConnectorMachine / iAP2 packet-link state machine
```

**Important boundary:** the endpoint above is proven to be runtime-supplied, but its concrete value is not. DIO's `/dev/ipod0` endpoint remains a separate evidence chain in TRACE-002.


```text
bluetooth
   |
   +--> Bluetooth iAP proxy
   |
   +--> iAP / iAP2 infrastructure
            |
            +--> [transport creation: unresolved]
            |
            +--> [DIO handoff: unresolved]
```

The existence of these components is proven; the arrows after the component boundary are not yet recovered as a function-level production call chain.

## Evidence

- E-021 — Bluetooth active-device callback reaches `CIapBTChannel::updateiAPDevice()`
- E-022 — runtime-supplied endpoint reaches `open64()`
- E-023 — `iap` has no `/dev/ipod0` literal; Bluetooth endpoint must not be equated with DIO's USB path without further evidence

- E-006 — production `enableIap=false`
- E-007 — Bluetooth-side iAP proxy exists

## Required next trace

Recover:

1. configuration parser and branch for `enableIap`;
2. caller/callee sequence into the Bluetooth iAP proxy;
3. iAP2 transport creation;
4. device/socket/IPC boundary;
5. callback/event handoff into DIO;
6. runtime confirmation against the same firmware baseline.

## Do not infer

- `enableIap=true` is sufficient;
- the lab `hci0` or `/dev/ttyS0` path is production HCI;
- presence of the iAP proxy proves Wireless CarPlay is active.
