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


## Newly recovered iAP2 transport boundary

The MU0678 ipod-drvr-iap2.so binary materially changes the interpretation of the DIO/iAP2 boundary.

The driver contains a generic transport dispatch layer:

~~~text
link_create()
 |
 +--> transport_get_link_params()

link_send_probe()
 |
 +--> transport_send_pkt()

transport_receive()
 |
 +--> transport-owned callback
~~~

It also contains separate Identify handlers for Bluetooth, Wi-Fi, USB device/host and serial transport components. The Wi-Fi descriptor contains TransportSupportsiAP2Connection and TransportSupportsCarPlay; the Bluetooth descriptor contains TransportSupportsiAP2Connection and BluetoothTransportMediaAccessControlAddress.

This proves that the iAP2 driver has a real multi-transport capability model. It does not prove that DIO can directly consume a wireless transport.

The shipped /etc/mm/iap2.cfg selects Lightning Connector, so the current production configuration remains USB-oriented despite the compiled wireless capability.

**Evidence:** E-032, E-033, E-034, E-035, E-036, E-037.


The shipped smartphone_integrator configuration independently reinforces the USB-side boundary: it monitors /dev/ipod0 and defines its CarPlay child as dio_manager. This is orchestration evidence, not proof that wireless iAP2 cannot be integrated later.

## New binary boundary recovered

A direct ARM disassembly of the MU0678 binaries resolves the previously unresolved argument/device boundary.

`dio_manager` has a `_ZTVN3dio15CIpodAP2ServiceE` vtable at `0x19bc58`. One recovered virtual target is `0x15e240`. Within that method:

~~~text
0x15e2c8  -> select path pointer into r9
0x15e2ec  mov r1, r9
0x15e2f0  mov r0, r11
0x15e2f4  bl  0x15ddb4
~~~

The helper at `0x15ddb4` then reaches the imported `iap2_connect` PLT entry at `0x115de8`:

~~~text
0x15ddb4:
    ...
    r1 = caller-supplied path
    r0 = output handle storage
    ...
0x15dde0:
    mov r0, r4
    bl  0x115de8   ; iap2_connect
0x15dde8:
    str r0, [r5]
~~~

At the callsite `0x15e2f4`, `r9` is passed as the first argument to the helper and `r2` has been set to `1`; the helper preserves the caller's second argument and therefore invokes `iap2_connect(path=r9, flags=1)`.

The path is therefore not compiled into `libiap2client.so`. It is supplied by the DIO-side caller.

This is the first direct binary edge that changes the previous interpretation of the DIO boundary: `/dev/ipod0` is a production configuration/default path, not an intrinsic requirement of the `iap2_connect()` client ABI.

The exact upstream construction of `r9` and whether the Bluetooth `CIapBTChannel` endpoint can reach this same DIO path remain unresolved.

**Evidence:** E-049, E-050, E-051. See TRACE-011.
