# TRACE-002 — DIO → iAP2 Transport

**Status:** Partial

## Objective

Determine whether DIO's iAP2 service is intrinsically tied to USB `/dev/ipod0` or consumes a transport abstraction.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | DIO iAP2 integration surface (`iAP2Connect` / `CIpodAP2Service`) |
| Process / binary | `dio_manager` plus iAP2 components |
| Caller / callee | `notifyiAP2DeviceConnected` / `iAP2Connect` / `CIpodAP2Service` are identified; `iap2_connect` and `iap2_disconnect` callsites are recovered in `dio_manager` |
| Arguments | Unresolved |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Unresolved |
| Device / socket / file boundary | Production `/dev/ipod0` is established; `iap2.device` configuration and DIO callsites are recovered, but exact lower-level open/use ownership remains unresolved |
| Protocol event | iAP2 event boundary not fully recovered |
| Runtime confirmation | Production `/dev/ipod0` use is established; complete DIO→transport runtime path is not |
| Evidence IDs | E-005, E-010, E-016 |
| Remaining uncertainty | Whether DIO can consume the separately recovered Bluetooth runtime endpoint or requires the `/dev/ipod0` service boundary |

Determine whether DIO's iAP2 service is intrinsically tied to the USB `/dev/ipod0` device or consumes a transport abstraction.

## Newly recovered binary trace

Production `dio_manager` contains:

```text
iap2.device = /dev/ipod0
iap2_connect  0x115de8
iap2_disconnect 0x115e00
```

Direct callsites recovered in the binary are:

```text
0x15dde0 -> iap2_connect
0x15dbfc -> iap2_disconnect
```

The production configuration also contains the CIpodAP2Service messages:

```text
[CIpodAP2Service] ... Connected to iAP2 driver, at: "%s"
[CIpodAP2Service] ... Connect to iAP2 driver at: "%s"
[CIpodAP2Service] ... Failed to connect to iAP2 driver at: "%s"
```

This strengthens the production DIO-side USB boundary from component inventory to a recovered configuration value and dynamic-call boundary. Separately, the Bluetooth `iap` binary now proves that its own iAP endpoint is runtime-supplied and passed to `open64()`. The two endpoints remain independent until the Bluetooth service's returned path is recovered and compared with `/dev/ipod0`.

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

## Required next trace

Recover the constructor/service setup, device open/use operations, transport object or callbacks, error handling, and DIO session transition.

## Cross-trace correction

The Bluetooth endpoint recovered in TRACE-001 must **not** be treated as `/dev/ipod0` merely because both paths implement iAP2. DIO proves `/dev/ipod0`; Bluetooth proves a runtime-supplied path. The equality or difference is an unresolved cross-process fact.

## Decision gate

Do not modify DIO or replace `/dev/ipod0` until the actual transport boundary is recovered.
