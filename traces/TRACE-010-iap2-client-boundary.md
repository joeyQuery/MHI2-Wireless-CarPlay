# TRACE-010 — MU0678 iAP2 Client / Media-Synchronizer Boundary

**Status:** Partial

## Objective

Determine whether libiap2client.so or mss-ipodiap2.so is the missing Bluetooth-to-DIO transport adapter, and establish the boundary represented by their exported iap2_connect() API.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | Exported iap2_connect() / iap2_disconnect() client API |
| Process / binary | libiap2client.so; mss-ipodiap2.so |
| Caller / callee | Exported iAP2 client API; exact production caller chain unresolved |
| Arguments | Caller-supplied state/arguments; exact production meaning unresolved |
| Return / error behaviour | Client-side status/error handling present; exact production semantics unresolved |
| IPC / ASI / DSI boundary | libiap2client.so imports QNX MsgSend, MsgSendv, MsgSendsv, MsgSendvs, plus open/close; destination/channel not recovered |
| Device / socket / file boundary | iap2cli independently exposes /dev/ipod0 as its default iPod mount/device argument; no Bluetooth/Wi-Fi selector recovered in libiap2client.so |
| Protocol event | iAP2 client API only; no Bluetooth/Wi-Fi packet transport event recovered here |
| Runtime confirmation | None for wireless path |
| Evidence IDs | E-045, E-046, E-047, E-048 |
| Remaining uncertainty | Exact callers, MsgSend destination/resource-manager path, and whether DIO CIpodAP2Service reaches this API directly or through another layer |

## Established

### 1. libiap2client.so is a client API, not a recovered Bluetooth transport selector

The MU0678 libiap2client.so exports the complete iap2_* client surface, including iap2_connect, iap2_disconnect, iap2_eap_open, iap2_eap_send, iap2_eap_recv, iap2_hid_start, iap2_hid_stop, iap2_medialib_*, iap2_raw_msg_send and iap2_vehicle_update.

Its imported system-facing functions include MsgSend, MsgSendv, MsgSendsv, MsgSendvs, open, close and ionotify.

No Bluetooth, Wi-Fi, HCI, socket, or transport-component symbol is imported by this library. Therefore the library cannot presently be identified as the missing Bluetooth transport adapter from its own binary interface.

This does not prove that the destination reached through QNX message passing is USB-only or that another process cannot supply a wireless endpoint.

### 2. mss-ipodiap2.so exposes the same iAP2 client API

mss-ipodiap2.so exports the same core iap2_* connection/service API, including iap2_connect, iap2_disconnect, iap2_raw_msg_send, iap2_eap_*, iap2_hid_* and iap2_medialib_* functions.

The recovered iap2_connect() and iap2_disconnect() functions have the same sizes and the same high-level instruction structure as the implementations in libiap2client.so, but their function bytes are not byte-identical. This is documented as a shared implementation shape, not as a byte-for-byte duplicate.

mss-ipodiap2.so identifies itself as DESCRIPTION=iPod iap2 Media Synchronizer and contains media/database synchronization code. Its role is therefore materially different from the Bluetooth CIapBTChannel implementation in /eso/bin/apps/iap.

### 3. devu-iap2ncm-tegra3-ci.so is concretely a USB device-controller driver

The MU0678 binary contains:

DESCRIPTION=Driver for ChipIdea USB OTG peripheral controller.
Featuring descriptors for an iAP2 NCM accessory.
io-usb-dcd -diap2ncm-tegra3-ci
/builds/workspace/PSP_USB_br650_be650SP1/svn/hardware/devu/dc/ci/chipidea.c

Its exported implementation is dominated by USB device-controller functions including chip_idea_usb_controller_methods, chip_idea_transfer, chip_idea_endpoint_init, chip_idea_get_descriptor, chip_idea_set_device_state, chip_idea_start, chip_idea_stop and io_usb_dll_entry.

This is direct evidence that the shipped NCM component is a USB device-controller implementation. It must not be treated as the wireless iAP2 bridge merely because its filename contains iap2ncm.

This does not rule out reuse of higher-level NCM/iAP2 code elsewhere; no such reuse was recovered here.

### 4. iap2cli confirms the normal client-side device convention

iap2cli links against libiap2client.so.1 and uses iap2_connect(). Its command-line help contains:

-m path -- specific iPod mountpoint (default: /dev/ipod0)

Thus /dev/ipod0 is a concrete client-side iAP2 device convention in the shipped image. This does not establish that every iap2_connect() call is hard-coded to /dev/ipod0, nor does it establish that the Bluetooth CIapBTChannel endpoint equals /dev/ipod0.

## Current trace

```text
Bluetooth iAP
    |
    v
CIapBTChannel::openiAPDevice()
    |
    v
open64(runtime-supplied endpoint)
    |
    v
[Bluetooth iAP2 endpoint]  <-- endpoint value unresolved
    |
    X  no recovered direct edge
    |
    +--> libiap2client.so::iap2_connect()  <-- client API only
    |
    +--> mss-ipodiap2.so                   <-- iPod iAP2 media synchronizer
    |
    +--> devu-iap2ncm-tegra3-ci.so         <-- USB device-controller code

DIO
    |
    v
CIpodAP2Service
    |
    v
iap2_connect()
    |
    v
/dev/ipod0 [production configuration]
```

The crossed edge is intentional: no binary evidence currently proves that the Bluetooth runtime endpoint is handed to libiap2client, mss-ipodiap2, or DIO.

## What this rules out

- devu-iap2ncm-tegra3-ci.so is the wireless CarPlay transport;
- mss-ipodiap2.so is the Bluetooth transport adapter;
- libiap2client.so itself selects Bluetooth/Wi-Fi transport;
- existence of iap2_connect() proves that Bluetooth iAP is already connected to DIO.

## Required next trace

1. Recover callers of libiap2client.so::iap2_connect() and iap2_disconnect().
2. Recover the QNX message destination/resource-manager path used by those client calls.
3. Recover the CIpodAP2Service constructor/service setup and determine whether it directly calls the client library.
4. Recover the Bluetooth iAP proxy endpoint publication path and compare its runtime endpoint with the DIO /dev/ipod0 path.
5. Only then determine whether a transport-neutral adaptation point exists between Bluetooth iAP2 and DIO.

## Do not infer

- MsgSend* means /dev/ipod0;
- iap2_connect() is itself a transport selector;
- mss-ipodiap2.so is the owner of Bluetooth iAP;
- the NCM driver can be reused wirelessly without a recovered caller/transport relationship.