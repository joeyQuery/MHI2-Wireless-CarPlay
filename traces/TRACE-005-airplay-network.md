# TRACE-005 — AirPlay Network Binding

**Status:** Partial

## Objective

Connect AirPlay interface-selection APIs to the actual socket/network interface used by the receiver.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | AirPlay network/interface setup |
| Process / binary | `libairplay.so` |
| Caller / callee | `_UpdateBonjourAirPlay` → `if_nametoindex()` → `DNSServiceRegister()` is recovered; substantive packet/multicast helper callers remain unresolved |
| Arguments | Bonjour registration receives `interfaceIndex`; exact packet/multicast helper arguments remain unresolved |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Unresolved |
| Device / socket / file boundary | Bonjour interface index is proven; actual packet/multicast socket interface remains unresolved |
| Protocol event | Bonjour/AirPlay networking APIs are present; production socket path not fully correlated |
| Runtime confirmation | No complete runtime proof of the selected interface is committed |
| Evidence IDs | E-008, E-009, E-018, E-019, E-020, E-026, E-031 |
| Remaining uncertainty | Callers/arguments of packet and multicast helpers, runtime interfaceName value, and final socket binding |

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

A second important boundary is now established: these DNS-SD calls are a process/library boundary into `libdns_sd.so` and ultimately `mdnsd`; see TRACE-003.

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
AirPlay object
   |
   +--> interfaceName (object + 0x6c)
          |
          +--> if_nametoindex(interfaceName)
                    |
                    +--> DNSServiceRegister(..., interfaceIndex, ...)
                              |
                              v
                         libdns_sd.so
                              |
                              v
                            mdnsd

Separately:
AirPlay
   +--> SocketSetPacketReceiveInterface() [substantive; caller/args unresolved]
   +--> SocketSetMulticastInterface()     [substantive; caller/args unresolved]
   +--> IsWiFiNetworkInterface()          [substantive; callers/args unresolved]
```


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

Trace callers and arguments of `SocketSetPacketReceiveInterface`, `SocketSetMulticastInterface`, and `IsWiFiNetworkInterface`; recover where the AirPlay `interfaceName` field is populated; then correlate the selected interface with runtime mDNS/AirPlay traffic.

## Decision gate

Do not equate presence of Wi-Fi-aware functions with a working Wireless CarPlay network path.
