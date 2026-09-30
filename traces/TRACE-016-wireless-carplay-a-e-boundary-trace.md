# TRACE-016 — Wireless CarPlay A–E Boundary Trace: Bluetooth Bootstrap, iAP2, CarPlay Trigger, and AirPlay/mDNS

Date: 2026-10-01

This trace records the findings established after the recent merge and the A–E follow-up campaign. It replaces several earlier broad assumptions with concrete MU0678 boundaries.

The scope is Wireless CarPlay only.

## Executive result

The MU0678 firmware already contains the low-level pieces required for the two transport halves:

- Bluetooth RFCOMM/iAP2 endpoint infrastructure in `btstack`.
- A Bluetooth endpoint consumer in `iap` / `CIapBTChannel`.
- Generic multi-transport iAP2 capability in `ipod-drvr-iap2.so`.
- Wi-Fi/AP networking and mDNS infrastructure.
- A production AirPlay receiver in `libairplay.so`.
- DIO Bluetooth/CarPlay state and session-management boundaries.

The missing functionality is therefore not a new Bluetooth driver or a replacement for the existing USB iAP2 path.

The traces establish three substantive missing orchestration layers:

1. Bluetooth bootstrap/control orchestration equivalent to the MH2p wireless iAP2 bootstrap.
2. `_carplay-ctrl._tcp` discovery and `GET /ctrl-int/1/connect` session triggering.
3. iAP2-over-AirPlay session messaging equivalent to MH2p `iAPSendMessage`.

---

## A — Bluetooth endpoint to iAP2 convergence

### A.1 Final MU0678 path

The recovered Bluetooth path is:

    Bluetooth phone
        |
        v
    btstack
        |
        +--> IapServices / RFCOMM SDP
        |
        +--> IapDevice resource manager
                |
                v
        IapDeviceServices
                |
                v
        iap / CIapBTChannel
                |
                +--> openiAPDevice()
                +--> connectToiAPDevice()
                +--> readiAPDevice()
                +--> writeiAPDevice()
                |
                v
            Bluetooth iAP2

The RFCOMM service channel and QNX endpoint pathname are separate data objects.

The RFCOMM channel is allocated through:

    0x23799c
        -> 0x242574
            -> 0x2081ac

and written into the iAP2 SDP record at `0x2d2519`.

The IapDevice pathname follows a different path:

    cbRf @ 0x23d500
        -> 0x23c028
            -> IapDevice creation case @ 0x23c720
                -> runtime string source sp+0x154 / sp+0x150
                -> string state sp+0x140
                -> IapDevice constructor @ 0x23fe24
                -> IapDevice +0x0c
                -> resmgr_attach @ 0x23fc84

The connected phone's Bluetooth MAC is independently stored by:

    0x23cd8c
        -> reconstruct BT address
        -> 0x23e394
        -> IapDevice +0x28/+0x2c

### A.2 The endpoint is not DIO's /dev/ipod0

MU0678's `iap` application has:

    CIapBTChannel::openiAPDevice()       0x109a9c
    CIapBTChannel::connectToiAPDevice()  0x109570
    CIapBTChannel::closeiAPDevice()      0x108d6c
    CIapBTChannel::readiAPDevice()       0x108c14
    CIapBTChannel::writeiAPDevice()      0x108bc8
    CIapBTChannel::updateiAPDevice()     0x10b434

The path is runtime-supplied through the IapDeviceServices boundary.

This is architecturally separate from DIO's existing:

    dio_manager
        -> iap2_connect()
        -> /dev/ipod0

The path-driven `libiap2client.so::iap2_connect()` ABI does not imply that Bluetooth IapDevice endpoints should be routed through the USB iAP2 driver.

### A.3 Closed conclusion

**The MU0678 Bluetooth bootstrap is a btstack IapDevice -> CIapBTChannel path, not a Bluetooth -> DIO /dev/ipod0 path.**

The exact final MU0678 pathname string is still not statically proven to have an MH2p-style BT-address suffix. The compiled `/dev/iapDevice` base participates in construction, but the final pathname comes from a runtime string-state path.

---

## B — iAP2 transport capability versus actual Bluetooth bootstrap ownership

MU0678 `ipod-drvr-iap2.so` contains real multi-transport iAP2 capability.

The feature startup path is:

    iap2_start_features()
        -> bt_send_info()
        -> bt_start_updates()

The packet dispatcher contains:

    link_handle_iap2pkt()
        -> bt_update_recv()
        -> wifi_acc_config_info()

The Wi-Fi accessory configuration handler implements the response to:

    RequestAccessoryWiFiConfigurationInformation

and supplies:

- SSID
- passphrase
- security type
- channel

WPA2 handling and error paths are also present.

The transport descriptors contain:

    Bluetooth:
        TransportSupportsiAP2Connection
        Bluetooth MAC

    Wi-Fi:
        TransportSupportsiAP2Connection
        TransportSupportsCarPlay

These are compiled capabilities.

They do **not** establish that MU0678 production Bluetooth bootstrap traffic is currently handed from `btstack` into `ipod-drvr-iap2.so`.

The existing `iap` application has its own raw endpoint and iAP2 link-layer path through `CIapBTChannel`.

### B.1 Closed conclusion

The correct distinction is:

    ipod-drvr-iap2.so
        = generic / driver-side multi-transport iAP2 capability

    iap / CIapBTChannel
        = MU0678 Bluetooth endpoint consumer

Do not replace the latter with the former merely because both contain iAP2 code.

---

## C — BluetoothSmartphoneIntegration and CarPlay activation

MU0678 contains a real Bluetooth/DIO CarPlay policy boundary.

Relevant DIO symbols include:

    BluetoothSmartphoneIntegration
    BluetoothSmartphoneIntegrationReply
    CBluetoothController
    notifyiAP2DeviceConnected
    notifyiAP2DeviceDisconnected
    iAP2Connect
    requestCarPlay
    checkCarPlayCompatibility
    onEvent_sessionCreated
    onEvent_sessionFinalized

MU0678 `bluetooth` also contains:

    bluetooth.enableIap
    SERVICETYPE_IAP2
    ERROR_CARPLAY_ACTIVE
    Carplay

The exact `bluetooth.enableIap` configuration mapping is already closed:

    CBluetoothTopologyReconnect constructor @ 0x155178
        -> configuration reader 0x155060
        -> 0x154c3c
        -> cached byte +0x28
        -> policy consumer 0x17b280

That property gates Bluetooth iAP permission/topology-reconnect policy.

It is **not** proven to be the btstack process-start switch or the direct IapDevice creation switch.

### C.1 Missing MU0678 bridge

No recovered MU0678 call chain establishes:

    CIapBTChannel
        -> WirelessCarPlayUpdate
        -> smartphone_integrator
        -> _carplay-ctrl._tcp
        -> dio_manager

nor an equivalent direct:

    WirelessCarPlayUpdate
        -> requestCarPlay()

The MH2p production reference has a distinct orchestration layer for this purpose. MU0678's BluetoothSmartphoneIntegration is therefore a policy/state boundary, not proof of the complete wireless bootstrap.

### C.2 Closed conclusion

**Bluetooth iAP infrastructure and DIO CarPlay state exist, but the production wireless-CarPlay orchestration bridge between them is absent from the recovered MU0678 path.**

---

## D — _carplay-ctrl._tcp discovery and session trigger

The MH2p wireless architecture contains:

    _carplay-ctrl._tcp
        -> controller discovery
        -> Bluetooth-MAC correlation
        -> CarPlayControlClient
        -> GET /ctrl-int/1/connect
        -> AirPlay session

The MU0678/MHI2-era receiver does not contain the corresponding `CarPlayControlClient` implementation or the recovered `_carplay-ctrl._tcp` browse/connect path.

MU0678 does contain normal AirPlay Bonjour machinery, including:

    _airplay._tcp
    DNSServiceRegister
    DNSServiceUpdateRecord
    DNSServiceQueryRecord
    DNSServiceGetAddrInfo

That is receiver advertisement/discovery infrastructure. It is not the MH2p wireless controller-trigger path.

### D.1 Closed conclusion

The missing sequence is:

    phone
        -> _carplay-ctrl._tcp
        -> receiver discovers controller
        -> BT-MAC correlation
        -> GET /ctrl-int/1/connect
        -> AirPlay session

It cannot be obtained merely by making `uap0` advertise `_airplay._tcp`.

---

## E — AirPlay interface selection and mDNS

### E.1 AirPlay Bonjour consumer

In MU0678 `libairplay.so`, the AirPlay object's `interfaceName` is consumed by:

    interfaceName
        -> if_nametoindex()
        -> DNSServiceRegister(... interface index ...)

The relevant object storage is `+0x6c`.

The production session setter functions:

    AirPlayReceiverSessionScreen_SetIFName
    AirPlayReceiverSessionScreen_SetTransportType
    AirPlayReceiverSessionScreen_SetClientIfMACAddr

are no-op stubs and must not be treated as the interface-selection mechanism.

### E.2 DIO AirPlay creation boundary

MU0678 `dio_manager` has:

    AirPlayReceiverServerCreate()       @ 0x13f0b4
    AirPlayReceiverServerSetDelegate()  @ 0x13f34c

and an active:

    R_ARM_GLOB_DAT
    AirPlayReceiverServerSetProperty
    @ 0x19da70

The GLOB_DAT relocation is real linkage.

However, a direct MU0678 callsite has not been recovered that proves the property arguments are:

    "interfaceName", <selected interface>

DIO does import `CFObjectSetPropertyCString` and has direct callsites, but those have not been proven to be the AirPlay server's `interfaceName` setter.

Therefore the correct status is:

    AirPlay interfaceName consumer        PROVEN
    AirPlay server creation               PROVEN
    SetProperty linkage                   PROVEN
    exact MU0678 interfaceName writer      NOT RECOVERED

### E.3 mDNS environment

MU0678 `dio_manager` contains:

    setEnv
    getEnvName
    putenv(%s): %s
    startMdnsdProcess
    restartMdnsd
    stopMdnsdProcess
    MDNS_DIRECTLINK_IFACE=carplay0

MU0678 `mdnsd` independently consumes the `MDNS_DIRECTLINK_IFACE` environment variable while building its interface list.

Thus:

    mdnsd consumes MDNS_DIRECTLINK_IFACE       PROVEN
    dio_manager has env-setting machinery       PROVEN
    exact DIO putenv -> mdnsd child edge         NOT RECOVERED

The presence of the string in DIO must not be treated as proof that the production process environment is set at a particular callsite.

### E.4 Closed conclusion

The mDNS/AirPlay mechanism itself is interface-aware. The unresolved static boundary is only the exact MU0678 producer that writes the AirPlay server's `interfaceName` and the exact producer that supplies the `mdnsd` environment.

---

# Combined architecture after TRACE-016

The evidence now supports:

    iPhone
      |
      +---------------- Bluetooth ----------------+
      |                                            |
      v                                            v
    btstack                                   iAP / CIapBTChannel
      |                                            |
      +-- RFCOMM SDP                               |
      +-- IapDevice -------------------------------+
      |                                            |
      +-- Bluetooth MAC                            |
                                                   v
                                            Bluetooth iAP2 bootstrap
                                                   |
                                                   | Wi-Fi configuration /
                                                   | wireless-CarPlay control
                                                   v
                                                iPhone
                                                   |
      +------------------- Wi-Fi ------------------+
      |
      v
    uap0 / AP
      |
      +--> mDNS
      |
      +--> AirPlay
              |
              v
          dio_manager
              |
              +--> CarPlay state/session
              |
              +--> AirPlay receiver
              |
              +--> iAP2-over-AirPlay
                    [missing MHI2-era equivalent]

The critical missing orchestration is between these already-present subsystems.

---

# Implementation implications

The traces rule out several tempting but incorrect approaches:

1. **Do not route Bluetooth IapDevice through DIO's existing `/dev/ipod0` path.**
   The Bluetooth endpoint is a separate resource-manager/iAP2 path.

2. **Do not treat `ipod-drvr-iap2.so`'s Bluetooth descriptor as proof of the production Bluetooth bootstrap.**
   It proves compiled capability, not current activation.

3. **Do not treat `BluetoothSmartphoneIntegration` as the missing wireless bootstrap.**
   It is a DIO Bluetooth/CarPlay policy boundary; the complete MH2p-style orchestration chain is not recovered.

4. **Do not implement wireless CarPlay by only enabling `_airplay._tcp` on `uap0`.**
   The MH2p `_carplay-ctrl._tcp` discovery and `/ctrl-int/1/connect` trigger are separate.

5. **Do not use the production Screen setter stubs as interface selectors.**
   The active AirPlay interface path is `interfaceName -> if_nametoindex -> DNSServiceRegister`.

6. **Do not infer the exact MU0678 AirPlay interface writer from the `AirPlayReceiverServerSetProperty` GLOB_DAT relocation alone.**
   The linkage is proven; the property call arguments are not.

---

# Trace status

| Trace | Status | Result |
|---|---|---|
| A | **Closed** | Bluetooth endpoint is `btstack IapDevice -> CIapBTChannel -> iAP2`, separate from DIO `/dev/ipod0`. |
| B | **Closed** | `ipod-drvr-iap2.so` has multi-transport capability; `iap` remains the recovered Bluetooth endpoint consumer. |
| C | **Closed** | Bluetooth/DIO CarPlay state exists, but no recovered MU0678 equivalent of the MH2p wireless orchestration bridge exists. |
| D | **Closed** | No recovered MU0678 `_carplay-ctrl._tcp` / `CarPlayControlClient` / `GET /ctrl-int/1/connect` path. |
| E | **Closed to producer boundaries** | AirPlay/mDNS consumers are proven; exact MU0678 `interfaceName` writer and exact DIO-to-mdnsd environment edge remain unproven static boundaries. |

## Remaining implementation-relevant boundaries

After A–E, the remaining work is specifically:

- Bluetooth iAP2 bootstrap/control-message semantics.
- Bluetooth-MAC identity propagation into the Wi-Fi/AirPlay side.
- `_carplay-ctrl._tcp` discovery and controller matching.
- `GET /ctrl-int/1/connect` implementation/trigger.
- AirPlay session establishment on the selected AP interface.
- iAP2-over-AirPlay message transport.
- Exact MU0678 `interfaceName` property producer.
- Exact MU0678 `mdnsd` environment producer.

RFCOMM channel allocation and `bluetooth.enableIap` provenance are no longer open targets; those were closed by the preceding trace work.
