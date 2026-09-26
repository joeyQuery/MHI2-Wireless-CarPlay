# TRACE-005 — AirPlay Network Binding

**Status:** Partial

## Objective

Connect the AirPlay interface-selection API surface to the actual socket/network interface used by the receiver.

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
