# TRACE-005 — AirPlay Network Binding

**Status:** Partial

## Objective

Connect AirPlay interface-selection APIs to the actual socket/network interface used by the receiver.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | AirPlay network/interface setup |
| Process / binary | `libairplay.so` |
| Caller / callee | Real packet/multicast interface helpers are identified; exported screen setters and `SocketSetBoundInterface` are stubs in production |
| Arguments | Unresolved |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Unresolved |
| Device / socket / file boundary | Socket/interface boundary is the target; actual production interface unresolved |
| Protocol event | Bonjour/AirPlay networking APIs are present; production socket path not fully correlated |
| Runtime confirmation | No complete runtime proof of the selected interface is committed |
| Evidence IDs | E-008, E-009, E-018, E-019, E-020 |
| Remaining uncertainty | Actual callers, interface comparisons, packet/multicast binding path and selected runtime interface |

Connect the AirPlay interface-selection API surface to the actual socket/network interface used by the receiver.

## Newly recovered binary trace

The pristine production `libairplay.so` separates the apparent interface API surface from the lower-level helpers:

- `AirPlayReceiverSessionScreen_SetIFName` — no-op stub.
- `AirPlayReceiverSessionScreen_SetTransportType` — no-op stub.
- `AirPlayReceiverSessionScreen_SetClientIfMACAddr` — no-op stub.
- `SocketSetBoundInterface` — tiny stub/constant-return.
- `SocketSetPacketReceiveInterface` — substantive implementation.
- `SocketSetMulticastInterface` — substantive implementation.
- `IsWiFiNetworkInterface` — substantive implementation.
- `IsUSBNetworkInterface` — constant-return helper.

Bonjour/mDNS support remains present, including `DNSServiceRegister`, `DNSServiceUpdateRecord`, `DNSServiceGetAddrInfo`, `DNSServiceQueryRecord`, and `_airplay._tcp.`.

Therefore the production AirPlay network trace must be followed through the real packet/multicast/interface helpers rather than assuming the screen setters or `SocketSetBoundInterface` perform the binding.

## Established

`libairplay.so` contains:

```text
SocketSetBoundInterface
SocketSetPacketReceiveInterface
SocketSetMulticastInterface
IsWiFiNetworkInterface
IsUSBNetworkInterface
```

It also contains Bonjour/mDNS APIs and `_airplay._tcp.`.

## Current trace

```text
AirPlay Screen
   |
   +--> SetIFName
   +--> SetTransportType
   +--> SetClientIfMACAddr
             |
             +--> [internal propagation: unresolved]
                         |
                         +--> SocketSetBoundInterface
                         +--> SocketSetPacketReceiveInterface
                         +--> SocketSetMulticastInterface
                                     |
                                     +--> [actual runtime interface: unresolved]
```

These are binary capabilities, not proof that the production MHI2 path uses `uap0`.

## Evidence

- E-008 — Bonjour/mDNS APIs
- E-009 — Wi-Fi/USB interface-selection helpers

## Required next trace

Recover the setter implementations/callers, argument values, interface-name comparisons and socket setup. Then confirm the runtime interface.

## Decision gate

Do not equate presence of Wi-Fi-aware functions with a working Wireless CarPlay network path.
