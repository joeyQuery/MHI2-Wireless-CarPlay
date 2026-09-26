# TRACE-002 — DIO → iAP2 Transport

**Status:** Partial

## Objective

Determine whether DIO's iAP2 service is intrinsically tied to the USB `/dev/ipod0` device or consumes a transport abstraction.

## Established

DIO contains:

```text
notifyiAP2DeviceConnected
notifyiAP2DeviceDisconnected
iAP2Connect
CIpodAP2Service
```

Production CarPlay exposes the USB-derived:

```text
/dev/ipod0
```

and iAP2-related components include:

```text
ipod-drvr-iap2.so
mss-ipodiap2.so
devu-iap2-tegra3-ci.so
devu-iap2ncm-tegra3-ci.so
libiap2client.so
```

## Current trace

```text
DIO
 |
 +--> notifyiAP2DeviceConnected / disconnected
 |
 +--> iAP2Connect
 |
 +--> CIpodAP2Service
          |
          +--> /dev/ipod0 [production USB boundary]
          |
          +--> [transport abstraction: unresolved]
```

The exact open/use sequence and whether `CIpodAP2Service` accepts an alternate transport have not been recovered.

## Evidence

- E-005 — production CarPlay uses `/dev/ipod0`
- E-010 — DIO contains iAP2 integration symbols
- E-015 — Wireless CarPlay is not yet proven end-to-end

## Required next trace

Recover the constructor/service setup, device open/use operations, transport object or callbacks, error handling, and DIO session transition.

## Decision gate

Do not modify DIO or replace `/dev/ipod0` until the actual transport boundary is recovered.
