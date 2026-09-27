# TRACE-011 — MU0678 DIO Runtime iAP2 Endpoint Path

**Status:** Partial

## Objective

Determine what DIO actually supplies to iap2_connect(), whether /dev/ipod0 is intrinsic to the client ABI, and whether the DIO boundary is structurally capable of consuming another QNX resource-manager endpoint.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | CIpodAP2Service vtable target 0x15e240 → helper 0x15ddb4 → imported iap2_connect at 0x115de8 |
| Process / binary | dio_manager → libiap2client.so |
| Caller / callee | 0x15e2f4 -> 0x15ddb4 -> 0x115de8 |
| Arguments | At 0x15e2f4, r1 receives path pointer r9; r2 is 1. Helper preserves caller r1/r2 and calls iap2_connect(path, 1) |
| Return / error behaviour | iap2_connect result is stored through the helper and tested by the caller |
| IPC / ASI / DSI boundary | iap2_connect calls open(path, flags), then sends a 20-byte control message with QNX MsgSend |
| Device / socket / file boundary | Current production path is /dev/ipod0; the recovered ABI accepts any caller-supplied path that open() can resolve |
| Protocol event | iap2_connect builds control message type 0x0113, length 19, field 0xffff9999, and sends it over the opened descriptor |
| Runtime confirmation | None for an alternate endpoint |
| Evidence IDs | E-049, E-050, E-051 |
| Remaining uncertainty | Exact upstream construction of DIO r9 path; whether Bluetooth CIapBTChannel endpoint can populate that path; exact resource-manager owner behind the alternate endpoint |

## 1. libiap2client::iap2_connect is path-driven

At 0x2cec:

~~~text
r5 = arg0
r6 = arg1

r0 = r5
r1 = r6
bl  0x114c        ; open

store returned fd

build 20-byte command:
  +0x00 = 0x0113
  +0x02 = 0x0014
  +0x04 = 0x0013
  +0x06 = 0xffff9999

MsgSend(fd, message, 20, reply, 4)

return connection handle
~~~

The first argument is therefore a path/device endpoint supplied by the caller. libiap2client.so does not hard-code /dev/ipod0.

## 2. DIO directly supplies that path

The DIO binary exports the C++ vtable symbol:

~~~text
_ZTVN3dio15CIpodAP2ServiceE @ 0x19bc58
~~~

A recovered vtable target is:

~~~text
0x19bc6c -> 0x15e240
~~~

Within 0x15e240:

~~~text
0x15e2c8  -> choose path pointer into r9
0x15e2ec     mov r1, r9
0x15e2f0     mov r0, r11
0x15e2f4     bl  0x15ddb4
~~~

The helper at 0x15ddb4 calls the imported iAP2 client:

~~~text
0x15ddb4:
  r4 = incoming r1
  ...
0x15dde0:
  mov r0, r4
  bl  0x115de8     ; iap2_connect
0x15dde8:
  str r0, [r5]
~~~

The caller has set r2 to 1 before this path, so the effective call is:

~~~text
iap2_connect(r9, 1)
~~~

The path is runtime/caller supplied.

## 3. Production /dev/ipod0 is configuration, not ABI

Production configuration still contains:

~~~text
iap2.device = /dev/ipod0
~~~

and smartphone integration monitors /dev/ipod0. Those are real production USB facts.

The new disassembly shows that the lower client ABI is not tied to that pathname. The correct model is:

~~~text
DIO configuration / runtime path
          |
          v
CIpodAP2Service
          |
          v
iap2_connect(path, 1)
          |
          v
open(path)
          |
          v
QNX resource-manager endpoint
          |
          v
iAP2 control message
~~~

This is materially different from treating /dev/ipod0 as an immutable DIO transport boundary.

## 4. Consequence for Wireless CarPlay

The Bluetooth-side trace already proves:

~~~text
CIapBTChannel
    -> open64(runtime-supplied endpoint)
~~~

TRACE-011 now proves the DIO/iAP2 side also accepts a runtime-supplied pathname at the client ABI.

Therefore the two sides are no longer separated by a pathname-level incompatibility. They may still be separated by endpoint ownership, protocol framing, iAP2 session state, or the way the Bluetooth endpoint is published.

This does not prove direct substitution. The next trace must identify:

1. the exact Bluetooth endpoint string returned by IapDeviceServices;
2. its QNX resource-manager owner;
3. whether that endpoint exposes the iAP2 command ABI expected by libiap2client.so;
4. whether the DIO caller can receive that endpoint instead of /dev/ipod0;
5. whether the same endpoint supports the complete DIO iAP2 event/session lifecycle.

## Do not infer

- /dev/ipod0 is the only possible iAP2 endpoint;
- any arbitrary file descriptor is an iAP2 endpoint;
- the Bluetooth endpoint is already compatible with libiap2client.so;
- Wireless CarPlay is now proven.

## Source evidence

MU0678 binary disassembly and ELF relocation/symbol evidence only. No external firmware is used for this trace.
