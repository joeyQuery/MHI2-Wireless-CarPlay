# TRACE-003 — MDNS_DIRECTLINK_IFACE

**Status:** Partial

## Objective

Recover where `MDNS_DIRECTLINK_IFACE=carplay0` is read and how its value reaches the mDNS/AirPlay network path.

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
