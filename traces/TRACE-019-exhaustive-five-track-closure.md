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
