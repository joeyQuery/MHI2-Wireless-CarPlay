# TRACE-012 — MU0678 iAP2 Client → Driver Resource-Manager Boundary

**Status:** Complete

## Objective

Determine whether libiap2client.so::iap2_connect() terminates at the shipped ipod-drvr-iap2.so resource-manager driver and recover the message ABI crossing that boundary.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | libiap2client.so::iap2_connect() @ 0x2cec |
| Process / binary | libiap2client.so → ipod-drvr-iap2.so |
| Caller / callee | DIO caller recovered in TRACE-011; client → driver handler matched by exact message ABI |
| Arguments | iap2_connect(path, flags); path is caller supplied |
| Return / error behaviour | open() returns fd; MsgSend expects 4-byte reply; driver replies with value 2 for the connect message |
| IPC / ASI / DSI boundary | QNX MsgSend → iap2_msg() → MsgReply |
| Device / socket / file boundary | open(path); exact production path remains /dev/ipod0, but pathname is not intrinsic to the client ABI |
| Protocol event | 20-byte connect message; halfword at +0x06 is 0xffff9999; driver checks +0x06 against 0x9999 |
| Runtime confirmation | None |
| Evidence IDs | E-053, E-054, E-055 |
| Remaining uncertainty | Exact mounted endpoint value and whether Bluetooth CIapBTChannel opens the same resource-manager service |

## 1. Driver object registration

The MU0678 ipod-drvr-iap2.so contains ipod_module at 0x27e7c:

```text
+0x00 = 0x2389c  -> "iap2"
+0x04 = 0x200
+0x20 = 0xa9bb0  -> iap2_drvr
```

The iap2_drvr object at 0xa9bb0 contains:

```text
+0x18 = iap2_init       @ 0xea6c
+0x20 = transport_recv_pkt @ 0x2121c
+0x24 = iap2_msg        @ 0xf4c8
+0x28 = iap2_notify     @ 0x1071c
+0x2c = iap2_unblock    @ 0xf368
+0x30 = iap2_ocb_calloc @ 0x10878
+0x34 = iap2_ocb_free   @ 0xf358
+0x38 = iap2_ocb_close  @ 0x107b4
+0x3c = fsys_open       @ 0x946c
+0x40 = fsys_read       @ 0x9260
+0x44 = fsys_lseek      @ 0x8c90
```

This is the concrete resource-manager driver object used by the shipped iAP2 module.

## 2. Client opens the caller-supplied endpoint

At libiap2client.so::iap2_connect() @ 0x2cec, arg0 is forwarded to the imported open() call at PLT 0x114c. The returned descriptor is retained as the QNX connection identifier.

No /dev/ipod0 literal is required by this function.

## 3. Exact 20-byte message ABI

The client constructs a 20-byte message and sends it with QNX MsgSend:

```text
+0x00 = 0x0113
+0x02 = 0x0014
+0x04 = 0x0013
+0x06 = 0xffff9999

MsgSend(fd, message, 20, reply, 4)
```

The MsgSend PLT entry is 0x102c.

## 4. Exact match in ipod-drvr-iap2.so

iap2_msg() @ 0xf4c8 loads the received halfword at message offset +0x06 and compares it with 0x9999.

On the match path it constructs reply value 2 and calls MsgReply() at PLT 0x4cf4 with a four-byte reply.

Thus the client and driver share an exact binary message ABI.

## 5. Resource-manager mounting

iap2_start_features() calls ipod_resmgr_mount(r9, 1), with r9 loaded from the iAP2 context at +0x14. The exact runtime mountpoint value is unresolved.

## 6. Proven edge

```text
DIO / libiap2client
        |
        | iap2_connect(path, flags)
        v
open(path)
        |
        v
QNX resource-manager endpoint
        |
        | MsgSend 20-byte connect ABI
        | +0x06 = 0xffff9999
        v
ipod-drvr-iap2.so::iap2_msg()
        |
        | MsgReply(..., 2)
        v
iap2_connect() returns connection handle
```

This replaces the former unresolved client-to-driver adaptation boundary with a concrete QNX resource-manager edge.

## 7. What remains

The next trace is no longer to find the DIO/client destination. It is to correlate:

```text
Bluetooth CIapBTChannel
    -> open64(runtime endpoint)
    -> endpoint owner
    -> same ipod-drvr-iap2 resource-manager?
```

Specifically recover the endpoint string returned by IapDeviceServices and determine whether it resolves to the same resource-manager service and accepts the same iAP2 message ABI.

## Do not infer

- Bluetooth endpoint = /dev/ipod0;
- Bluetooth endpoint already uses ipod-drvr-iap2;
- arbitrary QNX endpoint = iAP2;
- Wireless CarPlay is proven end-to-end.

## Source discipline

All findings come from MU0678 dump binaries and ELF/disassembly data only.
