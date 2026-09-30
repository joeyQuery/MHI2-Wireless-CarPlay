# MH2p Bluetooth Bootstrap and iAP2 Transport Path

How the MH2p (MIB2 High Plus, `MH2p_ER_AUG33_P2873`, QNX 6.6.0) carries iAP2 over Bluetooth, where the
Wireless CarPlay control messages are handled, and how iAP2 moves onto Wi-Fi/AirPlay once the session runs.
Provenance: [`firmware.md`](firmware.md). Evidence IDs P-300..P-326 (table at the end).

> **Scope rule.** Everything below marked *Proven* is proven **on MH2p only**, statically. It is
> cross-platform evidence of how a production VW Group / Alpine unit solves the problem, not proof about
> MHI2 (MU0678) or MHI2Q (MU1329). Paths are relative to `<MH2p>/` (see [README](README.md#paths)). Ghidra addresses carry
> Ghidra's +0x10000 image base (subtract 0x10000 for objdump/file offsets of `.text`/`.rodata`).

Related MH2p docs: [`configuration-and-boot.md`](configuration-and-boot.md) (P-100..P-123, config and boot
facts reused here), [`wifi.md`](wifi.md) (P-200..P-222, AP, credentials, Apple IE).

---

## 1. Summary

1. **iAP2 over Bluetooth is RFCOMM, owned by `btstack`.** `btstack` registers an SDP record with the iAP2
   accessory UUID `00000000-deca-fade-deca-deafdecacaff` on an RFCOMM channel, checks the iAP2 detect bytes
   `FF 55 02 00 EE 10` from the phone, and then publishes the RFCOMM byte stream as a **QNX resource-manager
   character device `/dev/iapDevice-<16 hex digits of the BT address>`** (P-301, P-303).
2. **The path travels by ASI, not by config.** `btstack` reports it with `updIapDevicePath(addr, path)` to the
   `bluetooth` app, which keeps it per device and publishes it in the `BluetoothMediaBridge` device list
   (`rfCommDeviceName`). Its clients are `smartphone_integrator` (SI), `iap` and (for music) `media`
   (P-305, P-306).
3. **Wireless CarPlay bootstrap does not run in `dio_manager`.** SI opens a connection in
   **`iap2connectionmanager`** (an SI child on ESO's own iAP2 stack `libesoiap2.so`, not Cinemo). Its
   `CIAP2TransportBT` just `open64(path, O_RDWR)`s the `/dev/iapDevice-*` node. It identifies with a Bluetooth
   transport component (id 99, `ESO-BT-Transport`, BT MAC) and a WirelessCarPlay transport component (id 98,
   `ESO-CoW-Transport`), receives `WirelessCarPlayUpdate`/`DeviceTransportIdentifierNotification`, and answers
   `RequestAccessoryWiFiConfigurationInformation` with SSID/passphrase/security/channel from `connectionmanager`
   (P-307..P-311).
4. **The phone is then found on Wi-Fi by Bonjour and matched by BT MAC.** SI browses `_carplay-ctrl._tcp` on
   `bcm0` and starts `dio_manager` for a Wi-Fi device. `dio_manager` runs Apple's `CarPlayControlClient`
   (in `libairplay.so`), compares `CarPlayControllerGetBluetoothMacAddress()` with the target phone's BT MAC,
   and calls `CarPlayControlClientConnect()` (P-313, P-314).
5. **After the session starts, iAP2 runs inside AirPlay.** `dio_manager` has a second iAP2 client,
   `CIAP2ServiceEso` (`libesoiap2`), whose transport (`CIAP2TransportHandler`) sends via
   `AirPlayReceiverSessionSendiAPMessage()` and receives the `iAPSendMessage` command. There is no separate
   AirPlay stream type for iAP2 (P-315, P-316).
6. **Cinemo has a pluggable iAP transport layer, and MH2p uses it for Bluetooth.**
   `libEsoIAPTransport.so` is a Cinemo transport plugin (`CreateEsoIAPOverBTTransport`, URL scheme `bt://`)
   loaded through `CINEMO_OPTION_IAP_TRANSPORT_LIBRARIES`. It opens the same `/dev/iapDevice-*` path and
   feeds A2DP audio in from the Bluetooth ASI. On MH2p only `media` uses it (iPod-over-Bluetooth), not
   `dio_manager` (P-317..P-319).
7. **MU1329's Cinemo has the same plugin hook**, with an older ABI: `IAP_TRANSPORT_LIBRARIES`,
   `ctli_context`, and `NmeTransport::Create(url, options, fn)`. MU1329's `btstack`, however, has no iAP
   SDP record, no iAP2 UUIDs, no detect check and no `/dev/iapDevice` resmgr. The Bluetooth half is missing
   there, not the Cinemo half (P-320..P-323).

---

## 2. Bluetooth side

### 2.1 Processes and configuration

| Item | MH2p fact | Source |
| --- | --- | --- |
| Launch | `connectivity_launcher` applications: `telephone`, `bluetooth`, `connectionmanager`, `messaging`, `srelay`, `btstack`, `nad`, `scon_audio`. There is **no `iap` entry** | `efs-system/etc/eso/production/connectivity.json` `launcher.applications` |
| BT transport | `btstack.transport = {type "uart", device "/dev/serbt1", baudrate 3000000}`; BCM4349B1 patchram (P-117) | `connectivity.json` `btstack` |
| iAP enable | `"bluetooth": { "enableIap": true }` | `connectivity.json` (P-107) |
| `bluetooth.conf` | `efs-system/etc/config/bluetooth.conf` is QNX's sample `io-bluetooth` file (`btp-btmgr.so`, `radiotype CSR_USB`). Nothing in the Alpine stack uses it | file content; the BT stack is `btstack` |
| Presets | `efs-system/etc/eso/presets/bluetooth*.json` hold log levels only (P-119) | presets |

**`enableIap` consumer (MH2p).** The `bluetooth` app reads the file-config keys `bluetooth.enableIap` and
`global.wirelessCarPlay` (strings `0x1238e4`, `0x123910`, Ghidra). They are parsed in `FUN_0006c094` and fetched
by `FUN_0006c318` / `FUN_0006c350`. `FUN_00061f5c` copies them into the `CBT_CONFIG` object (field +0x210 for
`enableIap`). The config dump names the fields `iapEnabled` and `wirelessCarplayEnabled` (`0x111608`,
`0x111624`). The iAP-specific logic lives in `CBluetoothIapManager`
(`connectivity/app_bluetooth/src/CBluetoothIapManager.cxx`):
- `FUN_0006fa44` walks the devices that have the iAP2 service bit `0x40000000`.
- It logs `iAP2 not allowed. Disconnect from 0x%012llX` or `iAP2 should be connected for ... -> connecting`,
  depending on an "allowed" check on that bit.
- `FUN_0006f4a0` handles the "WirelessCarplay should reconnect" last-mode reconnect
  (`LastConnectedCarplayAddress`, `LastmodeCarpalyActive`).

The exact branch that reads `CBT_CONFIG+0x210` was not followed to the "allowed" check (P-300).

### 2.2 `btstack`: SDP, RFCOMM and the device node

| Step | Evidence (MH2p `app/eso/bin/apps/btstack`) |
| --- | --- |
| SDP record | `.data` at file `0x1f9230`: ProtocolDescriptorList `35 0c 35 03 19 01 00 35 05 19 00 03 08 00` (L2CAP, RFCOMM, channel byte filled at run time), ServiceClassIDList `35 11 1c 00 00 00 00 de ca fa de de ca de af de ca ca ff` = **`00000000-deca-fade-deca-deafdecacaff`** (iAP2 accessory UUID). Registration logs `Registered iAP-SDP-Record. RfComm Channel: %d Status: %s` in `FUN_0015884c` (Ghidra); `IapServices.cxx` (`connectivity/btstack/btstack/iap/src/`) |
| Remote capability check | SDP search table at file `0x1f9090..`: HFP-AG, AVRCP, `00000000-deca-fade-deca-deafdecacafe` (iAP2 on the phone, `UUID_IAP2_HOST`), and a byte-reversed `2D8D2466-E14D-451C-88BC-7301ABEA291A` (`UUID_WIRELESS_CARPLAY_APPLE_DEVICE_REVERSE`). The forward UUID is also at `0x1a72d8` for the EIR check; `FUN_00071c34` logs `Wireless Carplay supported` and `iAP2 supported` |
| Incoming connection | `FUN_0015d74c`: `Incoming iAP service connection from %s`, `%d bytes coming in over rfcomm.`; `RF_AcceptChannel`/`RF_RejectChannel`; credit flow `CreditFlow enabled for 0X%012llX: %d, FrameSize: %d` |
| iAP2 detect | `FUN_0015d270` compares received bytes with the constant at `0x1e0940` = **`FF 55 02 00 EE 10`** (iAP2 link detect). On a match it sets a flag (+0x1b6), logs `Received correct iAP2 handshake`, and calls `FUN_0015cd1c`; otherwise it logs `not iAP2 compatible. Handshake-Answer: %s`. Retries: `IapCapabilityCheck failed after %d retries` |
| Device node | `FUN_0015cd1c` formats **`/dev/iapDevice-` + `%016llX`** (BT address), creates a `resmgr2` device (`common/resmgr2`, `Created device: %s`, `Got open request`, `Device already open!`, `Got read request - blocking client`) and calls `BluetoothServiceListener::updIapDevicePath(addr, filename)`. On teardown it calls `updIapDevicePath(addr, "")` |
| Stream semantics | Plain `io_read`/`io_write`: reads block until RFCOMM data arrives (`New data available`, `[%d] %d bytes coming in from resource`); writes go to `RF_SendData`. There is no message-framing ABI in the node |
| Wireless-CarPlay BT service | `WirelessCarPlayServices::connectService` logs `NOT SUPPORTED`. No second BT data channel exists for wireless CarPlay: Bluetooth carries only iAP2 |

This confirms P-106 by disassembly (P-301, P-302, P-303, P-304).

### 2.3 `bluetooth` app: from path to consumers

- `FUN_00046028` (`devicePath update for 0x%012llX: %s`) stores the path per BT address, only for devices
  with the iAP2 service bit `0x40000000`.
- `FUN_00070e10` (`Propagate 0x%012llX to media`) builds the device list (address, name, path, service flags
  `0x40000000`/`0x800000`) and pushes it to a listener.
- The `bluetooth` app is the **server** of `asi.connectivity.bluetooth.media.BluetoothMediaBridge`: only its
  binary contains `bluetooth/media/BluetoothMediaBridgeS.hxx`. Its clients are `smartphone_integrator`
  (`CBluetoothMediaBridgeWrapper`, log `[%i of %i] btAddress=0x%llx, btFriendlyName=%s, rfCommDeviceName=%s,
  type=%i (%s), flags=0x%x (%s)` in `FUN_000bed98`) and `iap` (P-305).
- `requestIAP2Reset` exists on both sides (SI `iap2ResetTimeout 7000`, P-102).
- The `bluetooth` app also serves `BluetoothSmartphoneIntegration`, the smartphone-mode interface.
  - It has modes `CARPLAY`, `CARPLAY_WIRELESS`, `ANDROID_AUTO_WIRELESS` and others, plus `Enabling wireless
    carplay for 0x%012llX` (`FUN_0007cc90`).
  - It handles OOB pairing: `Starting OOB-pairing for id %d`, `reply->iap2OOBBTPairingAccessoryInformation`,
    `iap2OOBBTPairingCompletionInformation`.
  - `dio_manager`'s `CBluetoothController` uses it to fetch the local BT address and set the smartphone mode
    (`Mode: %s successfully updated. iPhone's MAC address: %s`, `FUN_00093960`).

### 2.4 Proxies in `app/eso/lib/factories`

| Proxy | Interface names (strings) | Used by |
| --- | --- | --- |
| `libasimmxconnectivity_bluetoothproxy.so` | `asi.connectivity.bluetooth.BluetoothInformationService`, `.BluetoothSmartphoneIntegration` | `dio_manager` |
| `libasimmxconnectivity_bluetooth_mediaproxy.so` | `asi.connectivity.bluetooth.media.BluetoothMediaBridge` | SI, `iap` |
| `libasimmxconnectivity_bluetooth_a2dpproxy.so` | `asi.connectivity.bluetooth.a2dp.Media` | `libEsoIAPTransport.so` (A2DP audio for iAP-over-BT) |
| `libasimmxsmartphone_bluetoothproxy.so` | `asi.media.smartphonebluetooth.ISmartPhoneBluetooth` | SI (service header `ISmartPhoneBluetoothS.hxx`) |
| `libasimmxiap2connectionproxy.so` | `asi.media.iap2connection.IAP2Connection`; parameter structs `SConnectionParameter(USB)`, `SIAP2AccessoryWiFiConfigurationInformationParameter`, `SIAP2OOBBTPairingLinkKeyInformationParameter`, `SIAP2DeviceTransportIdentifierNotificationParameter`, `SIAP2WiFiInformationParameter`, ... | SI (client), `iap2connectionmanager` (server) |

MH2p has **no** `libasimmxconnectivity_bluetooth_iapproxy.so` (`IapDeviceServices`), which MU0678's
`CIapBTChannel` uses (E-015). MU1329 ships that proxy, but no MU1329 binary references `IapDeviceServices`.

---

## 3. `iap2connectionmanager`: the Bluetooth iAP2 owner for wireless CarPlay

| Aspect | MH2p fact | Source |
| --- | --- | --- |
| Parent | SI child `children.iap2connectionmanager {exec "iap2connectionmanager", path "/run/enforcer/eso/bin/apps", watchdogTimeout 10000}`; framework component id 396, `exec:null` (P-120) | `smartphone_integrator.json`, `stage2_ifs3/etc/eso/production/framework.json` |
| Stack | NEEDED `libesoiap2.so`, `libusbdi.so.2`; **no Cinemo**. Build path `mhd/iap2connectionmanager/../plugins/esoiap2` | ELF |
| ASI | Serves `asi.media.iap2connection.IAP2Connection` (`CIAP2ConnectionASIService`; methods in strings: `connect`, `disconnect`, `iap2IdentificationInformationUpdate`, `iap2CancelIdentification`, `iap2OOBBTPairingAccessoryInformation`, `iap2OOBBTPairingCompletionInformation`, `iap2RequestWiFiInformation`, `iap2AccessoryWiFiConfigurationInformation`, `iap2Start/StopPowerUpdates`, `iap2PowerSourceUpdate`). Up to 10 connections (`IAP2Connection_1..10`) | strings `0x431c0..0x435ec` |
| Events to SI | `Iap2WirelessCarPlayUpdate`, `Iap2DeviceTransportIdentifierNotification`, `Iap2StartOOBBTPairing`, `Iap2OOBBTPairingLinkKeyInformation`, `Iap2StopOOBBTPairing`, `Iap2WiFiInformation`, `Iap2RequestAccessoryWiFiConfigurationInformation`, identification/authentication results, `Iap2DeviceUUIDUpdate` and others | `CIAP2ConnectionEvent_*` strings `0x408dc..0x40c08` |
| Link parameters | `transport.usb.link` and `transport.bt.link` in `iap2connectionmanager.json`. BT: `maxNumberOfOutstandingPackets 5, maxReceivedPacketLength 2048, retransmissionTimeout 3000, cumulativeAckTimeout 700, maxNumberOfRetransmissions 30, maxCumulativeAcknowledgements 3`. USB: 5/4096/2000/100/30/3 | config (P-103) |
| BT transport | `CIAP2TransportBT` (`FUN_0004a21c`): copies the caller's path string and calls **`open64(path, 2 /* O_RDWR */)`**, logging `serv fd: %d` or `open of %s failed`. Reads use `select()` with a timeout (`No data within %d ms`) | Ghidra decompile |
| Identification | `FUN_00040b58` builds the transport components by connection type (`param+0x28`). See the list below | Ghidra decompile |
| Other modules | `CIAP2ModuleEAP` (`de.audi.mmiconnectdatatransfer`), `CIAP2ModuleNotifications` (`wirelessCarplayStatus=%i`, `bluetoothTransportIdentifier=%s`, `usbTransportIdentifier=%s`), `CIAP2ModuleOOBBTPairing`, `CIAP2ModulePower`, `CIAP2ModuleWiFiInformationSharing` | strings |

Transport components built by `FUN_00040b58`:
- type 2 (Bluetooth): **BluetoothTransportComponent id `0x63` (99)**, name `ESO-BT-Transport` (16 bytes), and the
  6-byte BT MAC from `param+0x2c`.
- type 0 (USB): **USBDeviceTransportComponent id `0x61` (97)**, `ESO-USB-Device-Transport`.
- If the caller's flag byte `param_3+6` is set, also a **WirelessCarPlayTransportComponent id `0x62` (98)**,
  `ESO-CoW-Transport` (17 bytes, "CarPlay over WiFi").

The ESO BT stack uses iAP2 link parameters of its own (3000 ms retransmit, 700 ms cumulative ACK) that differ
from the USB set (P-308). They are close to the Bluetooth profile in Apple's accessory spec, not to the
zero-ACK wired profile.

### 3.1 What SI does with it

- `CIAP2ConnectionWrapper` allows exactly one proxy (`only one iap2 connection proxy allowed`, `FUN_000b0be8`).
- It connects with `exlapNeeded=%u, enableWirelessCarPlayFeatures=%u` (`FUN_000b303c`); the identification is
  assembled there. The second flag is what switches the `ESO-CoW-Transport` component on (inference, from the
  matching flag in `FUN_00040b58`).
- `FUN_000b338c` sends **`iap2AccessoryWiFiConfigurationInformation`** with four fields:
  - SSID and passphrase, both from `WiFiAPInfoProvider`, i.e. `connectionmanager`'s `WlanService` (P-215).
  - Channel (`param+0x218`).
  - Security type mapped from the AP's WLAN type: 2,3 → 1; 4..9 → 2; else 0.
  - It logs `Successfully sent WiFi access point information for SSID '%s'`.
- The per-device gate in `FUN_000a7cb8`: `Device not connected via Bluetooth yet for deviceID=%i`,
  `5GHz access point is not available to activate Wireless CarPlay`, `Enabled Wireless CarPlay for
  deviceID=%i` (P-105, P-214).
- The device model holds `connectionType`, `macAddress`, `rfcommDeviceName`, `iapUUID` and USB identifiers
  (`FUN_000a54bc`, `FUN_000c0d28`, `FUN_000a4f20`: `rfcommDeviceName changed: %s -> %s`).
- `DeviceTransportIdentifierNotification` is used to merge the USB and Bluetooth identities of one phone:
  `Found existing deviceID=%i with same unique info as deviceID=%i`, `deviceID=%i is now known as
  deviceID=%i`.

---

## 4. Cinemo: the transport-plugin pattern (`libEsoIAPTransport.so`)

### 4.1 The plugin

`app/armle/usr/lib/cinemo/libEsoIAPTransport.so` (100,892 B, `DESCRIPTION=MIB2P-High Media Application
[plugins]`, 2021-07-09). NEEDED: `libcomm`, `libiplcommon`, `libosal`, `libutil`, `libcpp-ne.so.5`, `libc`,
`libm`. It is not listed in `cinemo_classes.xml`; Cinemo loads it as a transport library, not as a COM class.

Exported C ABI (`readelf --dyn-syms`, C++-mangled names, addresses are `.text` offsets):

| Symbol | Address | Role |
| --- | --- | --- |
| `CreateEsoIAPOverBTTransport` | (string `0x14b50`) | factory Cinemo calls; fills a `ctli_context` |
| `esoiapoverbt_send(void*, const void*, unsigned)` | `0xae69` | → `CEsoIAPOverBTTransport::send` (`write()` to fd) |
| `esoiapoverbt_receive(void*, void*, unsigned)` | `0xae25` | → `receive` (`select()` + `read()`) |
| `esoiapoverbt_abort(void*)` | `0xaead` | cancel blocking I/O |
| `esoiapoverbt_get_param / set_param(void*, CTLI_PARAM, ctli_params*)` | `0xaef1` / `0xaf35` | `CTLI_PARAM_PROTOCOL` returns `CTLI_PARAM_PROTOCOL_BLUETOOTH`; other params, e.g. `CTLI_PARAM_BLUETOOTH_MAC_ADDRESS` and the USB ones, log "not implemented for iAP over Bluetooth" |
| `esoiapoverbt_audio_create / delete / receive / abort / get_params / set_params` | `0xaf79` .. `0xb0cd` | A2DP audio: `CASIBluetoothA2DPMedia` connects `asi.connectivity.bluetooth.a2dp.Media`. `CA2DPStreamReceiver` gets PCM/codec frames through a QNX channel (`name_attach`/`name_open`, thread `A2DPStreamRcv_iAPoverBT`) |
| `esoiapoverbt_dataport_select / send / recv / cancel / enable` | `0xb111` .. `0xb1f1` | Cinemo data ports (native EA / HID), mostly stubbed |
| `esoiapoverbt_send_hid(void*, u16, const void*, unsigned)` | `0xcccd` | HID reports |
| `esoiapoverbt_delete_transport(void*)` | `0xadd1` | teardown |

- URL handling: `openDevicePath` accepts only **`bt://<devicePath>`** (`Invalid URL`, `Invalid devicePath.
  Cannot handle URL`).
- It takes options `bt_audio_file_node` and `prorities_streamrec_worker`
  (`devicePath=%s, a2dpStreamUrl=%s, a2dpStreamFilenode=%s`).
- It then does a plain `open()` of the path (`Opened %s successfully. fd=%i`) (P-317).

### 4.2 Who loads it and how

- **`media`** (not `dio_manager`) contains `iap://bt://`, `?bt_audio_file_node=`, `&prorities_streamrec_worker=`
  (`CBluetoothDetection`, strings `0x4d7d44..0x4d7e78`) and `:CreateEsoIAPOverBTTransport` (`CCinemoConfig`,
  `0x4dfb38`). It imports `CINEMO_OPTION_IAP_TRANSPORT_LIBRARIES` and `CINEMO_OPTION_IAP_TRANSPORT_OPTION_STRING`.
- `media.json` supplies the parts:
  - `media.cinemo.pluginDirPath "/armle/usr/lib/cinemo"` and `pluginNameIapTransport "EsoIAPTransport"`.
  - `media.bluetooth.iapSupport true`.
  - Under `player.ipod.accessory`: `bluetooth.EnableOOBBTPairing true` and `wifi.EnableWiFiInformationSharing
    true`, each commented `## Required for Wireless CarPlay`.
- The option value is therefore `"<pluginDirPath>/<pluginNameIapTransport>:CreateEsoIAPOverBTTransport"`
  (inference: the string is assembled from these two keys and the `:Create...` literal), and the media URL is
  `iap://bt:///dev/iapDevice-<addr>?bt_audio_file_node=...` (P-318).
- **`dio_manager` does not import the transport-library options.** Its only iAP2 URL is
  `iap2.device "iap://ffs:///dev/otg-cinemo"` (`dio_manager.json`, string `0xd1f08`). So on MH2p the Cinemo
  Bluetooth plugin serves **iPod/music over Bluetooth**, while CarPlay-over-Bluetooth bootstrap goes through
  `iap2connectionmanager`.

### 4.3 The mechanism inside Cinemo

`libNmeBaseClasses.so` (MH2p):
- `NmeTransportFactory::CreateTransport(ctli_create_params*, const INmeOptions&, NmeTransport*&)` walks the
  `IAP_TRANSPORT_LIBRARIES` entries (`Using IAP_TRANSPORT_LIBRARIES entry %d: "%s"`, `Skipping invalid entry
  in CINEMO_OPTION_TRANSPORT_LIBRARIES option: %s`) and `dlopen`s each (`dll_holder`).
- It calls `int create(ctli_create_params*, ctli_context*)`.
- It then validates every function pointer in the context (`Error in ctli_context: send function pointer is
  NULL`, ... `dataport_enable function pointer is NULL`).
- The stock `libNmeTransport.so` is itself such a library, exporting `CreateNmeIAPUSBTransport` (`usb`, host
  mode) and `CreateNmeIAPQRMTransport` (`ffs` device node, `eap_path`, `wait_device`).
- A library that cannot handle the URL's protocol returns an error (`Protocol "%s" cannot be handled.`).

**Reusable pattern for MHI2Q:** Cinemo selects the iAP transport from the URL scheme, and new schemes come from
a separately built shared library with a C entry point. That is how Alpine added Bluetooth to Cinemo without
changing Cinemo (P-319).

### 4.4 Cinemo iAP2 features for wireless (MH2p `libNmeVfs.so`)

Present on MH2p, absent from MU1329's `libNmeVfs.so` (string comparison):
- `Identify_Bluetooth_Transport`, `Identify_WIFI_Transport`, `WirelessCarPlayTransportComponent`,
  `OnWirelessCarPlayUpdate`.
- `NmeDDPIAPOOBBTResponder`, `IAPOOBBTPairing::LinkKeyInformation`, `OnStartOOBBTParing`,
  `IAPDevice::OOBBTPairingAccessoryInformation` / `CompletionInformation`, `BT_TRANSPORT_COMPONENT_ID`.
- `NmeDDPIAPWiFiResponder`, `IAPWiFiInfo::RequestAccessoryWiFiConfig`,
  `IAPDevice::AccessoryWiFiConfigurationInformation`, `IAPDevice::RequestWiFiInformation`.

MU1329 has only the Bluetooth half: `Identify_Transports(%s): can add Bluetooth transport only if
MAC-Address is given in configuration!`, `Bluetooth component name used is: '%s'`,
`BluetoothComponentInformation`, `OnBluetoothConnectionStatus` (P-322).

---

## 5. `dio_manager`: two iAP2 clients

| | `CIAP2ServiceCinemo` (Cinemo `libNmeSDK`) | `CIAP2ServiceEso` (`libesoiap2.so`) |
| --- | --- | --- |
| Transport | `iap://ffs:///dev/otg-cinemo` (USB device mode, NCM `carplay0` alongside, P-123) | `CIAP2TransportHandler` = iAP2 inside the AirPlay session |
| Components identified | `CinemoUSBComponent` (`FUN_000ae838`), **`CinemoBTComponent` with the local BT MAC** (`FUN_000aeb14`, log `BT component: transportName = %s, mac= %llu`), **`CinemoWiFiComponent`** (`FUN_000aee08`, `WiFi component: transportName = %s`), `VehicleInformationComponent` | built inside `libesoiap2` (control + file-transfer sessions; modules app launch, communications, EAP, media library, location, now playing, **WiFi information sharing**, identification, authentication; `FUN_000b70a0`) |
| Wireless messages handled | `StartOOBBTPairing`/`StopOOBBTPairing`, `OOBBTPairingLinkKeyInformation`, `OOBBTPairingAccessory/CompletionInformation` (`btTransportComponentId= %hd, deviceClass= %d`, `OOB BT Pairing result code: %d`), `OnWirelessCarPlayUpdate` → `CarPlay over WiFi, available: %d` (`FUN_000ade7a`), `DeviceTransportIdentifierNotification`; all relayed to SI (`CSIService`: `ISmartPhoneAppProxyReply::iap2WirelessCarPlayUpdate`, `iap2DeviceTransportIdentifierNotification`) | the same message set, for a session already on Wi-Fi |
| BT MAC source | `CBluetoothController` → `BluetoothSmartphoneIntegration::requestLocalBluetoothAddress`. If no answer within `bt.connectionTimeoutMs` 2000 ("required for having OOB BT pairing"), `s_btInterfaceTimerHandler` sends `bt.fakeMacAddr "AA:BB:CC:DD:EE:FF"` (`Sending fake MAC address: %s`) | — |

The iAP2-over-AirPlay code, by Ghidra decompile:
- **Send:** `FUN_0008769a` (job `CIAP2OverCarplayJob_SendIAP2Message` / `onJob_sendIAP2Message`) does
  `CFDataCreate(buf,len)` → **`AirPlayReceiverSessionSendiAPMessage(session, data, 0, 0)`**.
- **Receive:** `FUN_000b7b9a` (`CIAP2TransportHandler`) hands incoming buffers to the registered transport
  listener (the `libesoiap2` link), or logs `No transport listener!`.
- In `libairplay.so` (AirPlay 320.17.1) the channel is the AirPlay command **`iAPSendMessage`** (string
  `0xc3568`, API `OSStatus AirPlayReceiverSessionSendiAPMessage(AirPlayReceiverSessionRef, CFLDataRef, ...)`).
  There is no dedicated stream number.
- The `dio_manager` strings next to `iAPSendMessage` are the other AirPlay request names it handles:
  `startSession`, `disableBluetooth`, `hidSetInputMode`, `updateVocoderInfo`. `bluetoothIDs` and `deviceID`
  are keys; `bluetoothIDs` is also in `libairplay` (P-315, P-316).

Messages **not** found by name in any MH2p iAP2 stack (`libesoiap2`, `libNmeVfs`, `libNmeSDK`,
`dio_manager`, SI, `iap2connectionmanager`): `CarPlayAvailability` (0x4300) and `CarPlayStartSession`
(0x4301). The MH2p (2021, AirPlay 320) flow is the older one: Wi-Fi credentials over iAP2, then Bonjour
`_carplay-ctrl._tcp` and a CarPlayControl `/ctrl-int/1/connect` from the accessory (P-324).

---

## 6. Sequence as implemented on MH2p

Numbers in brackets are evidence IDs. *Inf* marks steps assembled from strings and structure rather than a
followed call chain.

1. **Boot.** `connectivity_launcher` starts `btstack` (UART `/dev/serbt1`) and `bluetooth`. `btstack` registers
   the iAP2 SDP record (UUID `…cacaff`, RFCOMM) [P-301]. SI starts `iap2connectionmanager` as its child
   [P-306]. `connectionmanager` brings up `bcm0` (5 GHz AP, 10.174.189.1) with the Apple vendor IE carrying
   the head unit's BT MAC [P-210, P-211].
2. **Pairing.** This is either normal SSP, or OOB over a wired iAP2 session:
   - The phone sends `StartOOBBTPairing`.
   - `dio_manager` (Cinemo, wired) gets the accessory OOB data from the `bluetooth` app
     (`iap2OOBBTPairingAccessoryInformation`), sends it to the phone, relays `OOBBTPairingLinkKeyInformation`
     back and completes [P-312, P-326].
   - During that wired session Cinemo also identified `CinemoBTComponent` and `CinemoWiFiComponent`, so the
     phone knows this accessory does wireless CarPlay. The phone answers `WirelessCarPlayUpdate` (0x4E0D) and
     `DeviceTransportIdentifierNotification` (0x4E0E), which reach SI [P-312, P-311].
3. **BT connect.** On a later start the `bluetooth` app reconnects the last CarPlay phone
   (`WirelessCarplay should reconnect`) [P-300].
   - `btstack` accepts the RFCOMM connection and verifies `FF 55 02 00 EE 10`.
   - It creates `/dev/iapDevice-<addr>` and calls `updIapDevicePath` [P-303].
4. **Path to SI.** The `bluetooth` app stores the path and publishes it in the `BluetoothMediaBridge` device
   list (`rfCommDeviceName`) [P-305].
5. **BT iAP2 session.** SI checks the gates: device known via Bluetooth, 5 GHz AP available, coding
   `carplayWireless` [P-310].
   - It calls `IAP2Connection::connect` with a Bluetooth `SConnectionParameter` (path + MAC) and
     `enableWirelessCarPlayFeatures` [P-309] (*Inf* for the exact parameter fields).
   - `iap2connectionmanager` opens the path [P-307], negotiates the BT link parameters [P-308], and
     authenticates (`/dev/i2cmfi`).
   - It identifies with BluetoothTransportComponent 99 and WirelessCarPlayTransportComponent 98 [P-307].
6. **Wi-Fi credentials.** The phone sends `RequestAccessoryWiFiConfigurationInformation` (0x5702). SI answers
   through `iap2AccessoryWiFiConfigurationInformation` (0x5703) with the `bcm0` SSID, passphrase, security and
   channel [P-309].
7. **Discovery.** The phone joins `bcm0` (DHCP from dnsmasq, P-116) and advertises `_carplay-ctrl._tcp`. SI's
   `CDNSServiceDiscoveryService` browses on `bcm0` (other interfaces are dropped, P-104), resolves the service,
   checks the MAC, and records a CarPlay Wi-Fi device [P-313].
8. **Session start.** SI starts `dio_manager` for connection type Wi-Fi, with `wifiInfo [ipAddress, macAddress,
   port, transportDeviceName]` (`FUN_00072b60`) and `localBtMacAddress` [P-313].
   - `dio_manager`'s AirPlay thread (`FUN_0008f8f8`) creates the AirPlay server on the Wi-Fi interface
     (the interface holding `wifi.ipaddr` 10.174.189.1, i.e. `bcm0`; P-110/P-217, decompiled in [airplay-dio.md](airplay-dio.md)), then creates and starts a `CarPlayControlClient`.
   - On a controller-added event (`s_carPlayControlClientEventCallback`) it compares
     `CarPlayControllerGetBluetoothMacAddress()` with the target BT MAC. On a match it calls
     `CarPlayControlClientConnect()` in a retry loop [P-314].
9. **AirPlay session.** The phone connects to the accessory's AirPlay server (pair-verify, SETUP; see the AirPlay
   docs).
   - iAP2 now runs over `iAPSendMessage` through `CIAP2ServiceEso` [P-315, P-316].
   - `dio_manager` sets the BT smartphone mode `CARPLAY_WIRELESS` via `BluetoothSmartphoneIntegration`
     [P-326]. The phone can also send the AirPlay request `disableBluetooth` (*Inf*: handler not traced).
10. **Teardown.** When the phone leaves the AP: `CarPlayControlClientSTALeft` (`FUN_0008e220`), SI
    `onEvent_carPlayWifiDeviceDisconnected`, and btstack `updIapDevicePath(addr, "")` when RFCOMM drops [P-314].

Open inside this sequence (see section 9):
- whether the BT iAP2 link in `iap2connectionmanager` stays up during the Wi-Fi session;
- how `media` and SI arbitrate the single-open `/dev/iapDevice-*` node (btstack: `Device already open!`).

---

## 7. The `iap` app on MH2p

`app/eso/bin/apps/iap` (182 KB, `connectivity/app_iap`, "MIB-High Unknown") is a small ESO iAP2-over-Bluetooth
client:
- transport component name `eso-Bt-iAP`;
- modules `IapAppWifi` / `CIapWifiBridge` (`asi.connectivity.networking.WlanService`, `New customer Ap details`,
  `ssid:`, `password:`, `Crypto:`, `channel:`);
- `IapAppItfBt` (`BluetoothMediaBridge` client: `Current active device: 0x%012llX (%s)`, `Setting local
  Bt-Address`), `IapAppItfCar` (VIN via `DSICarVehicleStates`), `IapAppFile`.

It is the MH2p descendant of MU0678's `iap`/`CIapBTChannel`, now fed by `BluetoothMediaBridge` instead of
`IapDeviceServices`. It is **not** in `connectivity_launcher.applications`, and no `framework.json` component
is named `iap`. Nothing found starts it, so it is dormant in this build (inference). The wireless CarPlay role
it might once have had now belongs to `iap2connectionmanager` (P-325).

---

## 8. Compared with MHI2Q MU1329 and MU0678

| Capability | MH2p (P2873) | MHI2Q MU1329 | MU0678 (per MHI2 repo docs) |
| --- | --- | --- | --- |
| `bluetooth.enableIap` | `true`; read by the `bluetooth` app into `CBT_CONFIG` (P-300) | `false` (`system/etc/eso/production/connectivity.json:216`); the `bluetooth` app has the key `bluetooth.enableIap`, the field `iapEnabled` and the enum `SERVICETYPE_IAP2` | `false`; consumer unresolved (E-006, Q1) |
| iAP2 SDP record / RFCOMM server in `btstack` | yes, UUID `…cacaff`, `IapServices` (P-301) | **absent**: no `…deca…` UUIDs, no `IapServices`/`iapDevice`/`FF5502` in MU1329 `btstack` (P-321) | not established in the repo |
| Remote wireless-CarPlay UUID check | yes, `2D8D2466-E14D-451C-88BC-7301ABEA291A` (P-302) | absent | — |
| BT iAP2 endpoint | `/dev/iapDevice-%016llX`, a `btstack` resmgr byte stream (P-303) | none | `CIapBTChannel` opens a runtime path from `IapDeviceServices` (E-021/E-022); value unknown |
| Bluetooth → consumers | `BluetoothMediaBridge` (`rfCommDeviceName`) → SI / `iap` / `media` | `IapDeviceServices` proxy present, unused; no `iap` app | `IapDeviceServices` → `iap` |
| iAP2 owner for wireless bootstrap | `iap2connectionmanager` (`libesoiap2`), SI child | none (no `iap2connectionmanager`) | `ipod-drvr-iap2.so` has BT/Wi-Fi components compiled in (E-033..E-035, E-040, E-041); activation unknown |
| Cinemo transport plugins | `IAP_TRANSPORT_LIBRARIES`, `ctli_create_params` ABI with data ports; `libEsoIAPTransport` for `bt://` | **present**, older `ctli_context` ABI (`Create(url, options, fn(ctli_context*, const char*, const char*))`, `cancel` instead of `abort`, `protocol_version`, no data ports); libraries: `libNmeTransport` (`usb`, `ffs`) only (P-320) | no Cinemo (`libiap2client` + `ipod-drvr-iap2`) |
| Cinemo wireless identification | `Identify_WIFI_Transport`, `WirelessCarPlayTransportComponent`, OOB, WiFi info sharing (P-322) | Bluetooth component only; no Wi-Fi/OOB/WiFi-info strings | n/a |
| `dio_manager` wireless iAP2 | `CIAP2ServiceEso` over `AirPlayReceiverSessionSendiAPMessage` (P-315) | no `libesoiap2`; no `iAPSendMessage` path; Cinemo USB only (separate MU1329 analysis) | `CIpodAP2Service` → `iap2_connect("/dev/ipod0")` (E-049..E-055) |
| CarPlayControl (`_carplay-ctrl`, connect by BT MAC) | `dio_manager` imports `CarPlayControlClient*`, `CarPlayControllerGetBluetoothMacAddress` (P-314) | MU1329 `libairplay` 210.81: absent (no `CarPlayControl` string) | not established |

---

## 9. Answers for the MHI2 repo

| STATUS / roadmap item | What MH2p shows (cross-platform evidence) |
| --- | --- |
| **Q1** `enableIap=false` branch | On MH2p the same key (`bluetooth.enableIap`) is **true**. It is a `bluetooth`-app file-config value stored in `CBT_CONFIG` (+0x210), next to `global.wirelessCarPlay`. The iAP logic it belongs to is `CBluetoothIapManager`: iAP2 service bit `0x40000000`, `iAP2 not allowed. Disconnect` / `iAP2 should be connected ... -> connecting`, and last-mode CarPlay reconnect [P-300]. So the flag belongs to the **`bluetooth` app's iAP2 connection policy**. It does not gate the RFCOMM implementation, which lives in `btstack`. MU1329 has the key but its `btstack` has no iAP implementation, so flipping it there has nothing to enable [P-321] |
| **Q2 / Q11** endpoint path from the Bluetooth service | On MH2p the endpoint is **`/dev/iapDevice-<16 hex digits of the BT address>`**, created by `btstack` after the iAP2 detect handshake and delivered as `rfCommDeviceName` / `updIapDevicePath` [P-303, P-305]. Inference for MU0678: its `CIapBTChannel` is the ancestor of MH2p's `iap` app. The MU0678 value is probably a `btstack` resmgr node of the same kind; check MU0678 `btstack` for `/dev/iapDevice-` and `IapServices` |
| **Q3** is it `/dev/ipod0`? | On MH2p **no**. The BT node is a separate resmgr owned by `btstack`, and the wired node is `/dev/otg-cinemo` [P-303, P-318]. Each transport has its own node; nothing makes Bluetooth pretend to be the USB node |
| **Q7** associating BT/iAP2 with Wi-Fi/AirPlay | Two links, both keyed on the phone's **Bluetooth MAC**:<br>(a) SI correlates USB and BT identities through `DeviceTransportIdentifierNotification` [P-311];<br>(b) `dio_manager` accepts a `_carplay-ctrl` controller only if `CarPlayControllerGetBluetoothMacAddress()` equals the target phone's BT MAC [P-314].<br>Also the head unit's BT MAC is in the Wi-Fi Apple IE (P-211) and in the iAP2 BT transport component (id 99) [P-307] |
| **Q9** activation branch into DIO | DIO is not activated by the BT iAP2 link directly. **SI** holds the BT iAP2 session (through `iap2connectionmanager`), sends Wi-Fi credentials, browses `_carplay-ctrl._tcp` on the AP, and then starts `dio_manager` with a Wi-Fi connection type and `wifiInfo` [P-309..P-313]. DIO's own iAP2 for the session runs over AirPlay [P-315] |
| **Q10** transport object after `open64()` | On MH2p the object is a **plain file descriptor on a byte-stream resmgr**. `CIAP2TransportBT` (`open64(path, O_RDWR)`, `select`/`read`/`write`) or `CEsoIAPOverBTTransport` (`open`, `select`, `read`, `write`) runs the iAP2 link layer itself on top of it [P-307, P-317] |
| **Q12** 20-byte `iap2_connect`/`iap2_msg` ABI on the BT endpoint? | **No, on MH2p.** The BT node carries raw iAP2 link bytes (`io_read` blocks for RFCOMM data, `io_write` → `RF_SendData`). No control-message ABI was found in `btstack`'s iAP resmgr [P-303]. The 0x9999 ABI of TRACE-012 belongs to `ipod-drvr-iap2.so`, which puts the iAP2 link *inside* the driver. The BT node is one layer lower (transport) |
| **Q13** reaching DIO when DIO is configured for `/dev/ipod0` | MH2p never hands the BT link to DIO: the bootstrap iAP2 ends in `iap2connectionmanager`/SI, and DIO gets iAP2 over AirPlay. For MHI2 this suggests a small **separate** BT iAP2 client (like `iap2connectionmanager`) that owns the BT link, rather than redirecting DIO's `/dev/ipod0` client to Bluetooth |
| Roadmap **2** (Bluetooth → iAP2) | "Trace Bluetooth iAP → iAP2 transport creation": the MH2p answer is `btstack` RFCOMM → resmgr node → ASI path → a userland iAP2 stack opening the node. "Production HCI transport": UART `/dev/serbt1` @ 3 Mbaud on MH2p (the MHI2 Marvell part uses SDIO) |
| Roadmap **3** (DIO iAP2 boundary) | "Is the service transport-neutral?" MH2p DIO has **two** iAP2 services, Cinemo for USB and ESO for AirPlay-tunnelled iAP2, not one neutral service. Cinemo itself is transport-neutral through `IAP_TRANSPORT_LIBRARIES` [P-319]. "Adaptation point for wireless iAP2": `AirPlayReceiverSessionSendiAPMessage` / `iAPSendMessage` [P-315, P-316] |
| TRACE-001 | The MH2p equivalent is complete statically: `btstack` `FUN_0015d270` → `FUN_0015cd1c` (`/dev/iapDevice-*`) → `updIapDevicePath` → `bluetooth` `FUN_00046028` / `FUN_00070e10` → `BluetoothMediaBridge` → SI → `iap2connectionmanager` `FUN_0004a21c` `open64` |
| TRACE-007 / TRACE-008 | MH2p shows the production form of MU0678's compiled-in BT/Wi-Fi components. Component ids 97 (USB device), 98 (CarPlay over Wi-Fi) and 99 (BT) exist; the Wi-Fi config response is filled from the AP's live credentials [P-307, P-309] |
| TRACE-011 / TRACE-012 | MH2p does not route the BT stream through a DIO-style client ABI. It supports the view that the 0x9999 ABI belongs to a USB-driver architecture that production wireless designs do not reuse |

---

## 10. What this means for MHI2 / MHI2Q wireless CarPlay

These are inferences; none of them is tested on MHI2 or MHI2Q.

1. **Bluetooth half (MHI2Q).** MU1329 `btstack` has no iAP2 RFCOMM server, which is the MH2p piece that
   matters most. Two options:
   - an out-of-`btstack` RFCOMM server, if the Blue SDK exposes RFCOMM/SDP registration to other processes
     (not established);
   - a replacement of that function.
   Either way it should end as a byte-stream endpoint like `/dev/iapDevice-*`.
2. **iAP2 client for the bootstrap.** MH2p runs the BT iAP2 session in a dedicated small process
   (`iap2connectionmanager`). It identifies BT and Wi-Fi transport components, answers 0x5702 with the AP
   credentials, and reports 0x4E0D/0x4E0E. MHI2Q has no such process, so an equivalent must be added. The
   open-source xcertplay/LIVI iAP2 implementations cover the same messages.
3. **Cinemo on MU1329 can take a transport plugin.** `IAP_TRANSPORT_LIBRARIES` and `ctli_context` exist in
   MU1329 `libNmeBaseClasses.so`, and the stock `libNmeTransport.so` is loaded the same way.
   - A `bt://`-style transport library (MH2p's `libEsoIAPTransport` design, rebuilt for MU1329's older ctli
     ABI) could let MU1329's own Cinemo run iAP2 over a BT node.
   - MU1329 Cinemo lacks the Wi-Fi/WirelessCarPlay identification and the 0x5702/0x5703 responders, so it
     would carry the link but not the wireless CarPlay messages. The bootstrap client still has to be ours.
   - The ABI difference is real: MU1329 is `create(ctli_context*, url, options)` with `cancel`, MH2p is
     `create(ctli_create_params*, ctli_context*)` with `abort` and data ports. `libEsoIAPTransport.so` cannot
     be dropped into MU1329 as-is.
4. **Session start.** MH2p does not start the Wi-Fi session from iAP2 (there is no 0x4301). It browses
   `_carplay-ctrl._tcp` and connects by BT MAC through `libairplay`'s `CarPlayControlClient`. MU1329's
   `libairplay` 210.81 contains no `CarPlayControl` and no `iAPSendMessage` string at all (checked while
   consolidating; see also [airplay-dio.md](airplay-dio.md) P-406/P-407), so MHI2Q cannot follow the MH2p
   flow with its stock library. It needs a different receiver either way, and could then use the MH2p flow
   or the newer 0x4300/0x4301 flow used by xcertplay.
5. **iAP2 after the session starts** travels in AirPlay `iAPSendMessage`. MU1329's `dio_manager` expects iAP2
   from Cinemo on `/dev/otg-cinemo`. Feeding it from AirPlay needs either a Cinemo transport plugin that
   bridges to the AirPlay command channel, or a separate client, as MH2p's `CIAP2ServiceEso` does.

---

## 11. Open / not determined

- The exact `SConnectionParameter` layout SI passes for a Bluetooth connection (type value 2 = BT in
  `iap2connectionmanager` `FUN_00040b58`; how SI fills path and MAC is not decompiled).
- Whether the BT iAP2 link in `iap2connectionmanager` is kept or closed after the AirPlay session starts, and
  who handles AirPlay `disableBluetooth` (string present in `dio_manager`, xref not resolved).
- Arbitration between `media` (Cinemo `iap://bt://…`) and SI (`iap2connectionmanager`) for the single-open
  `/dev/iapDevice-*` node; `requestIAP2Reset` exists on both sides.
- The exact code branch from `CBT_CONFIG.enableIap` (+0x210) to the iAP2 "allowed" check in
  `CBluetoothIapManager`.
- The RFCOMM channel number (filled at run time) and the SDP ServiceName attribute.
- How `media` receives the BT device (its `CBluetoothDetection` builds `iap://bt://`; the transport from the
  `bluetooth` app to `media` was not traced).
- ~~Whether MU1329 `libairplay.so` 210.81 exports `CarPlayControlClient*`~~: resolved while consolidating: it contains no `CarPlayControl` or `iAPSendMessage` string.
- MU0678 `btstack` was not examined here, so the `/dev/iapDevice-*` inference for Q2/Q11 is untested.

---

## Evidence

| ID | Finding | Class | Status | Source |
| --- | --- | --- | --- | --- |
| P-300 | `bluetooth` reads `bluetooth.enableIap` and `global.wirelessCarPlay` into `CBT_CONFIG` (`iapEnabled`, `wirelessCarplayEnabled`); `CBluetoothIapManager` enforces iAP2 allowed/connect and wireless-CarPlay reconnect | Static / Disassembly | Partial (field → branch not followed) | `app/eso/bin/apps/bluetooth` `FUN_0006c094`, `FUN_0006c318`, `FUN_00061f5c`, `FUN_0006fa44`, `FUN_0006f4a0`; `connectivity.json` |
| P-301 | `btstack` registers an iAP SDP record: ServiceClassID `00000000-deca-fade-deca-deafdecacaff`, L2CAP+RFCOMM | Static / Data / Disassembly | Proven | `btstack` file `0x1f9230..0x1f9253`; `FUN_0015884c` |
| P-302 | `btstack` SDP/EIR-checks the phone for iAP2 (`…cacafe`) and Wireless CarPlay `2D8D2466-E14D-451C-88BC-7301ABEA291A` | Static / Data | Proven | `btstack` `0x1f90bc`, `0x1f913f` (reversed), `0x1a728c`, `0x1a72d8`; `FUN_00071c34` |
| P-303 | After `FF 55 02 00 EE 10` is seen on RFCOMM, `btstack` creates resmgr `/dev/iapDevice-%016llX` (byte stream) and calls `updIapDevicePath(addr, path)` | Disassembly | Proven | `btstack` `FUN_0015d270`, `FUN_0015cd1c`, `FUN_0015d74c`; constant `0x1e0940` |
| P-304 | `WirelessCarPlayServices::connectService` is `NOT SUPPORTED`: no BT data channel beyond iAP2 | Static | Proven (string) | `btstack` `0x1a542c..0x1a548c` |
| P-305 | `bluetooth` stores the path per device (iAP2 bit `0x40000000`) and serves `BluetoothMediaBridge` whose device list carries `rfCommDeviceName`; SI and `iap` are clients | Static / Disassembly | Proven (server/client), Partial (list content) | `bluetooth` `FUN_00046028`, `FUN_00070e10`, `BluetoothMediaBridgeS.hxx`; SI `FUN_000bed98` |
| P-306 | `iap2connectionmanager` is an SI child on `libesoiap2` (no Cinemo), serving `asi.media.iap2connection.IAP2Connection` | Static / Config | Proven | ELF NEEDED; `smartphone_integrator.json`; strings `0x43518` |
| P-307 | `CIAP2TransportBT` does `open64(path, O_RDWR)`. Identification: BT component id 99 `ESO-BT-Transport` + MAC, USB device id 97 `ESO-USB-Device-Transport`, WirelessCarPlay id 98 `ESO-CoW-Transport` (flag) | Disassembly | Proven | `iap2connectionmanager` `FUN_0004a21c`, `FUN_00040b58` |
| P-308 | BT iAP2 link params 5/2048/3000/700/30/3 vs USB 5/4096/2000/100/30/3 | Config | Proven | `iap2connectionmanager.json` |
| P-309 | SI connects with `enableWirelessCarPlayFeatures` and sends `iap2AccessoryWiFiConfigurationInformation` {SSID, passphrase, channel, security 0/1/2} | Disassembly | Proven (send), Partial (connect parameters) | SI `FUN_000b303c`, `FUN_000b338c`, `FUN_000b0be8` |
| P-310 | SI enables wireless CarPlay per device only if the phone is BT-connected and the 5 GHz AP is available | Static | Proven (strings, function) | SI `FUN_000a7cb8` |
| P-311 | SI merges USB/BT identities of a phone via `DeviceTransportIdentifierNotification`; the device model holds `rfcommDeviceName` | Static | Partial | SI strings `0xee2a0..0xee35c`, `FUN_000a4f20`, `FUN_000a54bc` |
| P-312 | `dio_manager` Cinemo (wired) identifies USB, BT (local MAC) and Wi-Fi components and handles OOB BT pairing, WirelessCarPlayUpdate and DeviceTransportIdentifierNotification, relaying them to SI | Disassembly / Static | Proven (components, relay strings) | `dio_manager` `FUN_000ae838`, `FUN_000aeb14`, `FUN_000aee08`, `FUN_000ade7a`, `FUN_000ad90e`; `CSIService` strings |
| P-313 | SI browses `_carplay-ctrl._tcp` and starts `dio_manager` for a Wi-Fi device with `wifiInfo [ipAddress, macAddress, port, transportDeviceName]` | Static / Config | Partial | SI `CDNSServiceDiscoveryService`, `FUN_000db7ec`; `dio_manager` `FUN_00072b60`; P-100 |
| P-314 | `dio_manager` creates the AirPlay server and a `CarPlayControlClient`, and connects only to the controller whose BT MAC matches the target | Disassembly | Proven | `dio_manager` `FUN_0008f8f8`, `s_carPlayControlClientEventCallback` (0x8f0d8), `FUN_0008e220` (`STALeft`) |
| P-315 | `dio_manager` sends iAP2 via `AirPlayReceiverSessionSendiAPMessage`; `CIAP2TransportHandler` feeds received bytes to the `libesoiap2` link of `CIAP2ServiceEso` | Disassembly | Proven | `dio_manager` `FUN_0008769a`, `FUN_000b7b9a`, `FUN_000b70a0` |
| P-316 | iAP2 over AirPlay is the `iAPSendMessage` command, not a stream type | Static | Proven (string/API) | `eso/lib/libairplay.so` `0xc3568`, `0xc3b98` |
| P-317 | `libEsoIAPTransport.so` is a Cinemo iAP transport plugin (`CreateEsoIAPOverBTTransport`, `esoiapoverbt_*`) for `bt://<path>` with A2DP audio from ASI | Static (exports, strings) | Proven | `app/armle/usr/lib/cinemo/libEsoIAPTransport.so` |
| P-318 | `media` loads it through `CINEMO_OPTION_IAP_TRANSPORT_LIBRARIES` and uses `iap://bt://…`; `dio_manager` does not (USB `iap://ffs:///dev/otg-cinemo` only) | Static / Config | Proven (imports, strings, config); option value Inferred | `media`, `media.json` (`pluginNameIapTransport`, `bluetooth.iapSupport`, `EnableOOBBTPairing`, `EnableWiFiInformationSharing`), `dio_manager.json` |
| P-319 | Cinemo's `NmeTransportFactory` loads iAP transports from `IAP_TRANSPORT_LIBRARIES` and validates the `ctli_context` function table; `libNmeTransport` exports `CreateNmeIAPUSBTransport` and `CreateNmeIAPQRMTransport` | Static | Proven | `app/armle/usr/lib/libNmeBaseClasses.so`, `cinemo/libNmeTransport.so` |
| P-320 | MU1329 Cinemo has the same `IAP_TRANSPORT_LIBRARIES` mechanism with an older `ctli_context` ABI and only the `usb`/`ffs` transports | Static | Proven (strings/symbols) | `<MU1329>/app/armle/usr/lib/libNmeBaseClasses.so`, `libNmeSDK.so`, `cinemo/libNmeTransport.so` |
| P-321 | MU1329 `btstack` has no iAP SDP record, iAP2/wireless UUIDs, detect bytes or `/dev/iapDevice`. MU1329 `bluetooth` has `bluetooth.enableIap` and `SERVICETYPE_IAP2`; `connectivity.json` sets `false` | Static / Config | Proven (absence by byte search) | MU1329 `app/eso/bin/apps/{btstack,bluetooth}`, `system/etc/eso/production/connectivity.json:216` |
| P-322 | MH2p Cinemo `libNmeVfs` has Wi-Fi/WirelessCarPlay transport identification, OOB BT pairing and Wi-Fi info-sharing responders; MU1329 has only the Bluetooth component | Static | Proven (strings) | `libNmeVfs.so` MH2p vs MU1329 |
| P-323 | MU1329 ships `libasimmxconnectivity_bluetooth_iapproxy.so`, but no MU1329 app references `IapDeviceServices`; no `iap`, no `iap2connectionmanager` | Static | Proven (grep) | MU1329 `app/eso/lib/factories`, `app/eso/bin/apps` |
| P-324 | No `CarPlayAvailability`/`CarPlayStartSession` names in any MH2p iAP2 stack; the flow is Wi-Fi config + `_carplay-ctrl` + CarPlayControl connect | Static | Proven (absence of names) / Inferred (flow) | string search of `libesoiap2`, `libNmeVfs`, `libNmeSDK`, `dio_manager`, SI, `iap2connectionmanager` |
| P-325 | MH2p `iap` app is an ESO iAP2-over-BT client (`eso-Bt-iAP`, Wi-Fi AP sharing) with no launcher entry | Static / Config | Proven (strings, launcher config) / Inferred (dormant) | `app/eso/bin/apps/iap`; `connectivity.json`; `framework.json` |
| P-326 | The `bluetooth` app serves `BluetoothSmartphoneIntegration` (modes incl. `CARPLAY_WIRELESS`, OOB pairing replies); `dio_manager` `CBluetoothController` uses it and falls back to `bt.fakeMacAddr` after 2000 ms | Static / Disassembly | Proven (strings, call sites) | `bluetooth` `FUN_0007cc90`; `dio_manager` `FUN_00093960`, `s_btInterfaceTimerHandler`; `dio_manager.json` `bt` |
