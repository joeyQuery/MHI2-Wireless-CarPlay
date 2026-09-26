# TRACE-003 — MDNS_DIRECTLINK_IFACE

**Status:** Partial

## Objective

Recover where `MDNS_DIRECTLINK_IFACE=carplay0` is consumed and how the value reaches mDNS/AirPlay network binding.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | `MDNS_DIRECTLINK_IFACE` configuration |
| Process / binary | Configuration consumer not yet identified; relevant AirPlay/mDNS components are known |
| Caller / callee | Configuration key → consumer → interface propagation → socket binding remains unresolved |
| Arguments | Interface value is `carplay0` in production configuration; downstream arguments unresolved |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Unresolved |
| Device / socket / file boundary | `carplay0` is established as USB-derived network interface; downstream socket binding unresolved |
| Protocol event | Bonjour/mDNS APIs are present; production CarPlay discovery path on the configured interface is not fully traced |
| Runtime confirmation | Configuration value is established; consumer/runtime propagation is not |
| Evidence IDs | E-003, E-004, E-008, E-009, E-017 |
| Remaining uncertainty | Consumer process and actual interface binding |

Recover where `MDNS_DIRECTLINK_IFACE=carplay0` is read and how its value reaches the mDNS/AirPlay network path.

## Newly recovered binary trace

In production `dio_manager`, configuration metadata contains:

```text
MDNS_DIRECTLINK_IFACE=carplay0
```

at virtual address `0x190d2c`, adjacent to other mDNS configuration metadata. This establishes the value as part of the binary's configuration data rather than an isolated architectural description.

The consumer and propagation path are still not recovered. Therefore the trace remains:

```text
MDNS_DIRECTLINK_IFACE=carplay0
        |
        +--> [consumer: unresolved]
                    |
                    +--> [interface propagation: unresolved]
                                |
                                +--> [socket binding: unresolved]
```

## Established

Production configuration contains:

```text
MDNS_DIRECTLINK_IFACE=carplay0
```

The production USB CarPlay network is:

```text
USB -> devnp-usbdnet.so -> carplay0
```

AirPlay contains Bonjour/mDNS APIs and explicit network-interface helpers.

## Current trace

```text
MDNS_DIRECTLINK_IFACE=carplay0
        |
        +--> [configuration consumer: unresolved]
                    |
                    +--> [interface value propagation: unresolved]
                                |
                                +--> [socket/interface binding: unresolved]
                                            |
                                            +--> libairplay / Bonjour path
```

The configuration value itself is proven. The downstream chain is not.

## Evidence

- E-004 — production `MDNS_DIRECTLINK_IFACE=carplay0`
- E-008 — AirPlay exposes Bonjour/mDNS APIs
- E-009 — AirPlay exposes interface-selection helpers

## Required next trace

Search every consumer/reference, identify parser and process, recover the variable/object carrying the interface name, then connect it to socket setup or AirPlay configuration.

## Decision gate

Do not conclude that changing the value to `uap0` is sufficient until the consumer and downstream binding are known.
