# TRACE-019 — Exhaustive Five-Track Wireless CarPlay Closure Pass

Date: 2026-10-01

This trace records the exhaustive static pass over five requested Wireless CarPlay tracks:
1. Bluetooth to iAP2 convergence
2. Bluetooth-MAC propagation and session identity
3. MU0678 wireless-CarPlay control-plane equivalent
4. AirPlay interface selection and interfaceName
5. iAP2 0x5702 to 0x5703 Wi-Fi configuration activation

## Executive result

| Track | Result | Proven | Remaining boundary |
|---|---|---|---|
| 1 | Low-level endpoint closed; CarPlay convergence unresolved | btstack owns IapDevice, QNX resource manager, iAP2/RFCOMM SDP and IapDeviceServices; iap/CIapBTChannel consumes a runtime endpoint | Exact IapDeviceServices pathname serialization and production handoff into CarPlay activation |
| 2 | MAC storage closed; end-to-end correlation open | BT MAC is stored in IapDevice +0x28/+0x2c; iAP2 BT Identify has a MAC field; DIO has BT identity APIs | BT MAC to Wi-Fi/AirPlay/DIO session identity |
| 3 | MU0678 equivalent not recovered | No recovered _carplay-ctrl._tcp, /ctrl-int/1/connect, CarPlayControlClient or AirPlay-Receiver-Device-ID implementation | Wireless session-start trigger |
| 4 | Consumer closed; writer unresolved | interfaceName +0x6c -> if_nametoindex -> DNSServiceRegister; DIO property callback is real | Property-object identity and actual interfaceName value |
| 5 | Protocol handler closed; activation unresolved | link_handle_iap2pkt -> wifi_acc_config_info; 0x5702 response machinery produces SSID/passphrase/security/channel | Live production transport selection and AP credential source |

## 1. Bluetooth -> iAP2

The exact MU0678 btstack ELF proves:
- btstack::IapDevice vtable at 0x2d0a58
- resource-manager construction at 0x23fe24
- resmgr_attach at 0x23fc84
- compiled /dev/iapDevice base at file offset 0x1bf790
- iAP2 accessory SDP record at 0x2d250c
- RFCOMM UUID 0x0003
- iAP2 accessory UUID 00000000-deca-fade-deca-deafdecacaff
- IapServices registration/deregistration
- IapDeviceServices service/proxy/reply machinery

The proven low-level chain is:

    btstack
      -> IapServices
      -> iAP2/RFCOMM SDP
      -> IapDevice
      -> QNX resource-manager endpoint
      -> IapDeviceServices
      -> iap / CIapBTChannel
      -> open64(path)

IapServices registration helper 0x23799c reads its embedded +0x0c channel/control member, writes its first byte to SDP byte 0x2d2519 and calls shared SDP routine 0x20be48. The producer of IapServices +0x0c byte zero is behind shared btstack event machinery: 0x2379f0 passes +0x10/+0x0c to 0x242574, which reaches 0x2081ac. No direct local byte store was recovered.

The connected six-byte Bluetooth address is reconstructed from event/device +0x0d..+0x12 and reaches 0x23e394, which stores it in IapDevice +0x28/+0x2c.

The pathname is separate:
- sp+0x154 = pathname source pointer
- sp+0x150 = pathname length
- IapDevice +0x0c = pathname consumed by resmgr_attach

No /dev/iapDevice- literal or recovered address-to-string formatting operation was found in this path. Therefore the MH2p /dev/iapDevice-<BT address> convention remains reference-only for MU0678.

IapDeviceServices exposes updateActiveDevices, updateLocalBtAddress, updateLocalFriendlyName, clientConnected and clientDisconnected. The exact generated IPC serialization carrying the endpoint pathname into CIapBTChannel is not exposed as a unique symbol in the stripped image.

### enableIap correction

MU0678 bluetooth.enableIap is mapped to CBluetoothTopologyReconnect +0x28. The topology/reconnect consumer is 0x17b280, reached from 0x17b598, 0x17b938 and 0x17baf4. The branch performs reconnect/topology work and, on the disabled arm, consults FEC application state.

An exhaustive sweep of the inspected bluetooth executable found no CIapBTChannel, IapDeviceServices, updIapDevicePath, /dev/iapDevice-, concrete RFCOMM-server creation, or endpoint open API. Thus enableIap is a Bluetooth topology/reconnect policy input, not proven endpoint creation or btstack startup.

## 2. Bluetooth MAC propagation

Three MU0678 identity surfaces are proven:
1. btstack stores the connected phone BT address in IapDevice +0x28/+0x2c.
2. ipod-drvr-iap2.so has ident_info_tspbt and a Bluetooth descriptor containing TransportSupportsiAP2Connection and BluetoothTransportMediaAccessControlAddress.
3. dio_manager contains BluetoothSmartphoneIntegration, CBluetoothController, getBTMACAddress, responseLocalBluetoothAddress and setBluetoothMACAddrs.

What is not recovered is the cross-transport join:

    phone BT MAC
      -> Wi-Fi CarPlay controller identity
      -> AirPlay session identity
      -> same DIO CarPlay session

No MU0678 implementation equivalent to the MH2p CarPlayControllerGetBluetoothMacAddress matching path or SI DeviceTransportIdentifierNotification session merge was recovered.

Therefore BT-MAC storage and transport advertisement are closed; BT-MAC-to-Wi-Fi/AirPlay session correlation is not.

## 3. Wireless-CarPlay control plane

The inspected MU0678 evidence contains no recovered implementation of:
- _carplay-ctrl._tcp
- /ctrl-int/1/connect
- AirPlay-Receiver-Device-ID
- CarPlayControlClient
- CarPlayControllerGetBluetoothMacAddress

MU0678 does contain AirPlay receiver/server/session machinery and _airplay._tcp registration, but the MH2p wireless controller bootstrap is not present in the recovered MU0678 surface.

This does not prove that iOS requires that exact MH2p mechanism. Another AirPlay-generation-specific trigger could exist. However, no alternative MU0678 wireless session-start mechanism has been recovered.

The actual gap is therefore the wireless session-start/control-plane trigger, not the existence of AirPlay itself.

## 4. AirPlay interface selection

Pristine MU0678 libairplay.so proves:

    AirPlay object +0x6c
      -> interfaceName
      -> if_nametoindex()
      -> DNSServiceRegister(... interfaceIndex ...)

The following production exports are no-op stubs and are not active selectors:
- AirPlayReceiverSessionScreen_SetIFName
- AirPlayReceiverSessionScreen_SetTransportType
- AirPlayReceiverSessionScreen_SetClientIfMACAddr

DIO has direct AirPlayReceiverServerCreate at 0x13f0b4 and AirPlayReceiverServerSetDelegate at 0x13f34c.

AirPlayReceiverServerSetProperty is a real R_ARM_GLOB_DAT relocation at 0x19da70. The calls at 0x13f128 and 0x13f1f0 load that GOT entry and feed it through CFObjectSetPropertyCString/CFObjectSetProperty into the real libairplay property dispatcher.

The earlier interpretation that the GLOB_DAT was dead linkage is therefore disproven.

However, the recovered property argument is not proven to be the interfaceName property object. The temporary string at the recovered invocation is empty, and the property object comes from DIO read-only data whose identity has not been established as interfaceName.

Therefore the remaining AirPlay question is:

    DIO property-object construction
      -> property identity = interfaceName?
      -> selected interface value = uap0?
      -> libairplay object +0x6c

DIO does not import SocketSetPacketReceiveInterface, SocketSetMulticastInterface, SocketSetBoundInterface or IsWiFiNetworkInterface. Those helpers therefore are not DIO-level direct calls.

## 5. iAP2 0x5702 -> 0x5703

MU0678 ipod-drvr-iap2.so contains executable feature startup:

    iap2_start_features()
      -> bt_send_info()
      -> bt_start_updates()

The packet dispatcher contains:

    link_handle_iap2pkt()
      -> 0x13818 -> bt_update_recv()
      -> 0x1382c -> wifi_acc_config_info()

wifi_acc_config_info handles RequestAccessoryWiFiConfigurationInformation and contains response construction for:
- SSID
- passphrase
- security type
- channel

WPA2/WPA2-Personal handling and configuration-error paths are present.

Thus the 0x5702 protocol handler is closed as compiled MU0678 capability.

What remains unproven is the production activation/data-source chain:

    live Bluetooth iAP2 session
      -> 0x5702
      -> wifi_acc_config_info
      -> MU0678 AP credentials
      -> iPhone joins uap0

The MU0678 image independently contains uap0, connectionmanager AP machinery and the iAP2 Wi-Fi transport descriptor, but no recovered function-level production edge joins those pieces to a live Wireless CarPlay bootstrap.

## Cross-track result

The available evidence now reduces the architecture to:

    iPhone
      |
      +-- Bluetooth
      |     -> btstack IapDevice/RFCOMM
      |     -> IapDeviceServices
      |     -> iap / CIapBTChannel
      |     -> Bluetooth iAP2
      |          |
      |          +-- [Wi-Fi bootstrap activation unresolved]
      |          +-- [BT identity correlation unresolved]
      |
      +-- Wi-Fi
            -> uap0
            -> mDNS
            -> AirPlay
                 |
                 +-- [interfaceName writer unresolved]
                 +-- [wireless session-start trigger unresolved]
                 |
                 v
               DIO
                 |
                 +-- CarPlay session
                 +-- session-side iAP2 path unresolved

## What is now ruled out

1. Bluetooth iAP endpoint = /dev/ipod0 is not proven and must not be assumed.
2. bluetooth.enableIap is not the low-level endpoint creator; it belongs to topology/reconnect policy.
3. MU0678 lacks a Bluetooth iAP2 endpoint is disproven.
4. The compiled /dev/iapDevice string does not prove the MH2p BT-address suffix.
5. The three screen interface/transport setters are not active Wi-Fi selectors.
6. AirPlayReceiverServerSetProperty GLOB_DAT linkage is not dead.
7. That GLOB_DAT path proves interfaceName is set is not established.
8. The existence of wifi_acc_config_info does not prove a live 0x5702 exchange.
9. BluetoothSmartphoneIntegration does not prove Bluetooth iAP2 packets already reach DIO.
10. uap0 plus AirPlay primitives does not prove Wireless CarPlay.

## Exhaustion boundary

The five tracks have been exercised through every implementation-relevant edge exposed by the currently available MU0678 artifacts. Further progress now requires at least one of:
- the generated IapDeviceServices RPC serialization/active-device payload;
- a runtime Bluetooth/iAP2 capture;
- the missing AirPlay property-object producer;
- a runtime mDNS/AirPlay session capture;
- another binary outside the inspected MU0678 image containing the missing wireless orchestration.

The remaining implementation surfaces are therefore precisely isolated as:
A. Bluetooth bootstrap activation
B. BT-MAC identity correlation
C. wireless session-start/control plane
D. AirPlay interfaceName population
E. live 0x5702/0x5703 activation and credential source

Everything below those boundaries that the available MU0678 binaries expose has been traced as far as the artifacts permit.


## Recursive follow-up: the 0x9999 ABI is not the Bluetooth convergence layer

The five-track follow-up exposed an important architectural question: can MU0678 simply point DIO's existing `iap2_connect()` at the Bluetooth `IapDevice` pathname?

The evidence says **do not do that**.

MU0678's `libiap2client.so::iap2_connect()` opens its caller-supplied pathname and then sends the fixed 20-byte QNX control message containing the `0x9999` marker. The matching receiver is `ipod-drvr-iap2.so::iap2_msg()`, which recognizes the same marker and replies with 2.

That ABI is therefore specifically the client-to-`ipod-drvr-iap2.so` resource-manager boundary.

The Bluetooth `IapDevice` is a separate btstack resource-manager endpoint. The MH2p production reference proves the corresponding architectural distinction: its Bluetooth node carries raw iAP2 link bytes, while the userland Bluetooth bootstrap client performs the iAP2 link layer on top of the byte stream. DIO's session-side iAP2 is then carried through AirPlay, not by redirecting the USB iAP2 client to the Bluetooth node.

This is cross-platform reference evidence for the architectural distinction; MU0678-specific btstack read/write internals were not directly recoverable through the current GitHub binary interface. Therefore the precise MU0678 statement is:

**Proven:** the 0x9999 ABI is the `libiap2client` ↔ `ipod-drvr-iap2.so` boundary.

**Proven:** the Bluetooth endpoint is owned by btstack and consumed by `CIapBTChannel`.

**Not proven:** that MU0678's Bluetooth endpoint itself implements the 0x9999 ABI.

Consequently, redirecting DIO's `iap2_connect()` from `/dev/ipod0` to the Bluetooth endpoint is not supported by the evidence and should not be the implementation strategy.

## Recursive follow-up: the likely missing component class is now constrained

The available evidence contains a dedicated MU0678 `iap` process with `CIapBTChannel`, but no recovered production launcher edge proving when that process is started. The MH2p descendant is explicitly a small userland iAP2-over-Bluetooth client, and the MH2p wireless bootstrap uses a separate client of this type.

Therefore the unresolved MU0678 question is no longer "how do we make the Bluetooth resource-manager endpoint speak iAP2?" It already exposes the Bluetooth transport endpoint and a userland consumer.

The remaining question is whether the existing MU0678 `iap` process contains enough **Wi-Fi provisioning / WirelessCarPlay control logic** to be the bootstrap client, or whether a separate bootstrap process is required.

No recovered MU0678 evidence currently establishes:
- `WirelessCarPlayUpdate` handling in `iap`;
- `DeviceTransportIdentifierNotification` handling in `iap`;
- a live 0x5702 request originating from `iap`;
- an AP credential provider connected to `iap`;
- a wireless session-start controller.

That narrows the implementation search to the existing `iap`/Bluetooth service boundary and the iAP2 driver transport objects before introducing a new process.

## Recursive follow-up: the AirPlay control-plane gap is genuine

A repository-wide search for the MH2p receiver-side control-plane signatures found no MU0678 implementation of:

- `_carplay-ctrl._tcp`
- `CarPlayControlClient`
- `CarPlayControllerGetBluetoothMacAddress`
- `GET /ctrl-int/1/connect`
- `AirPlay-Receiver-Device-ID`
- `AirPlayReceiverSessionSendiAPMessage`
- `iAPSendMessage`

The MH2p reference shows these are not cosmetic helpers: they provide the wireless controller discovery, BT-MAC correlation, session trigger, and session-side iAP2 tunnel.

Therefore the absence is implementation-relevant. MU0678 cannot be made Wireless CarPlay merely by enabling its existing AirPlay Bonjour server.

A different AirPlay-generation protocol could theoretically replace this sequence, but no MU0678-specific substitute was recovered.

## Recursive follow-up: 0x5702 is a responder, not yet a bootstrap

The MU0678 `wifi_acc_config_info()` handler is real and constructs the expected 0x5703 fields. However, exhaustive repository searches did not recover a MU0678 equivalent of the MH2p credential-provider/orchestrator chain:

`connectionmanager WlanService → WiFiAPInfoProvider → bootstrap iAP2 client → 0x5703`.

The presence of `uap0` and the handler therefore answers only the protocol question: **MU0678 knows how to answer the request.**

It does not answer the activation question: **which live Bluetooth iAP2 client sends the request and supplies the AP credentials?**

That producer/activation edge remains genuinely absent from the recovered MU0678 evidence.

## Recursive follow-up: current five-track answers

1. **Bluetooth → iAP2:** the Bluetooth transport endpoint and its consumer are real. The existing USB/DIO 0x9999 client ABI is not the proven Bluetooth transport ABI.
2. **BT MAC:** the phone MAC is stored in btstack, but the recovered MU0678 evidence does not carry it through to Wi-Fi/AirPlay session selection.
3. **Control plane:** the MH2p wireless controller/session-start mechanism is absent from the recovered MU0678 surface; no replacement has been proven.
4. **AirPlay interface:** the consumer and property-dispatch path are real; the exact `interfaceName` property writer/value remains unresolved.
5. **0x5702/0x5703:** the responder is compiled and functional at the protocol level; its live bootstrap/credential source is not connected to the production Bluetooth path in the recovered evidence.
