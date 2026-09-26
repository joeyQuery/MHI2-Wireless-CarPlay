# TRACE-001 — Bluetooth → iAP2

**Status:** Partial

## Objective

Recover the production Bluetooth → iAP2 bootstrap path used for CarPlay-capable device establishment.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | Bluetooth CarPlay/iAP bootstrap entry; exact function not recovered |
| Process / binary | `bluetooth`; `btstack`; `libasimmxconnectivity_bluetooth_iapproxy.so`; iAP2 components |
| Caller / callee | Bluetooth-side iAP registration/proxy relationship is established at component level; exact caller/callee chain unresolved |
| Arguments | Unresolved |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Unresolved |
| Device / socket / file boundary | Transport/device boundary unresolved |
| Protocol event | Bluetooth/iAP2 protocol transition not yet captured as one production trace |
| Runtime confirmation | Component/runtime observations exist, but no complete timestamped Bluetooth→iAP2 execution trace is committed |
| Evidence IDs | E-006, E-007, E-014, E-015 |
| Remaining uncertainty | `enableIap` control flow, IapDeviceServices endpoint publication, Bluetooth transport handoff and DIO handoff |

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

The companion `libasimmxconnectivity_bluetooth_iapproxy.so` identifies `asi.connectivity.bluetooth.iap.IapDeviceServices`, `PROXY_asi_connectivity_bluetooth_iap_IapDeviceServices`, `STUB_asi_connectivity_bluetooth_iap_IapDeviceServices`, `IapDeviceServicesRPCStub`, and `IapDeviceServicesRPCStubReply`.

This establishes a concrete Bluetooth-iAP IPC/service boundary. It does not yet establish the owning Bluetooth process, endpoint publication, or the handoff from this service into DIO.

## Current trace

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
