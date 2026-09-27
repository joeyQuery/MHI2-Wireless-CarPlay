# TRACE-008 — MU0678 Wireless iAP2 Control Plane and DIO Bluetooth Integration

**Status:** Partial

## Objective

Record the additional MU0678-only findings that connect the compiled iAP2 multi-transport machinery to Bluetooth/Wi-Fi control messages and the DIO Bluetooth smartphone-integration boundary, without claiming that production Wireless CarPlay is enabled.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | `ipod-drvr-iap2.so` iAP2 packet dispatcher / feature startup |
| Process / binary | `ipod-drvr-iap2.so`; `dio_manager` |
| Caller / callee | `iap2_start_features()` → Bluetooth/Wi-Fi feature handlers; `link_handle_iap2pkt()` → `bt_update_recv()` / `wifi_acc_config_info()`; DIO contains `BluetoothSmartphoneIntegration` and `CBluetoothController` |
| Arguments | Bluetooth/Wi-Fi feature message objects and transport-specific state are present; exact production wireless session arguments remain unresolved |
| Return / error behaviour | Feature handlers and unsupported/configuration error paths are present; exact production wireless activation result remains unresolved |
| IPC / ASI / DSI boundary | DIO exposes Bluetooth Smartphone Integration through connectivity interfaces; exact Bluetooth iAP2-to-DIO handoff is unresolved |
| Device / socket / file boundary | `iap2.cfg` selects Lightning Connector; Bluetooth iAP endpoint remains runtime-supplied and unresolved; DIO separately uses `/dev/ipod0` |
| Protocol event | Bluetooth connection updates and accessory Wi-Fi configuration messages are implemented in the iAP2 driver |
| Runtime confirmation | Static MU0678 binary/configuration evidence only; no end-to-end wireless runtime trace |
| Evidence IDs | E-039, E-040, E-041, E-042, E-043 |
| Remaining uncertainty | How the existing wireless-capable components are activated in production, which transport object is selected, and how the selected iAP2 path is handed into DIO/AirPlay |

## iAP2 feature startup

The MU0678 driver contains executable feature-startup paths for Bluetooth and Wi-Fi-related iAP2 functionality:

```text
iap2_init()
    ├── link_create()
    ├── link_setup()
    └── iap2_send_probe()

iap2_start_features()
    ├── bt_send_info()
    ├── bt_start_updates()
    ├── HID
    ├── Device Audio
    ├── Now Playing
    ├── Media Library
    └── Telephony
```

This is stronger than static capability strings alone: the binary contains executable Bluetooth feature-startup machinery.

## Shared iAP2 packet dispatcher

The recovered `link_handle_iap2pkt()` path directly reaches both transport-specific control handlers:

```text
link_handle_iap2pkt()
    ├── bt_update_recv()
    └── wifi_acc_config_info()
```

Recovered call targets include:

```text
0x13818  -> bt_update_recv()
0x1382c  -> wifi_acc_config_info()
```

Therefore the iAP2 packet dispatcher contains first-class handling for both Bluetooth connection-status information and accessory Wi-Fi configuration information.

This does **not** prove that the production MHI2 currently routes a live Wireless CarPlay session through those handlers.

## Accessory Wi-Fi configuration response

The MU0678 driver contains a concrete handler for:

```text
RequestAccessoryWiFiConfigurationInformation
        ↓
SSID
Passphrase
SecurityType
Channel
        ↓
AccessoryWiFiConfigurationInformation
```

The implementation also contains explicit WPA2 / WPA2 Personal handling and configuration-error paths.

This establishes that the shipped iAP2 implementation contains an accessory-Wi-Fi provisioning/control mechanism.

It does not prove that the current production configuration exposes that mechanism to an iPhone.

## DIO Bluetooth smartphone integration

The MU0678 `dio_manager` binary contains:

```text
BluetoothSmartphoneIntegration
BluetoothSmartphoneIntegrationReply
CBluetoothController
```

with explicit CarPlay mode/state values alongside Android Auto, MirrorLink and NOTHING.

Recovered controller operations include:

```text
getBTMACAddress
responseLocalBluetoothAddress
responsePrepareConnect
reportConnectionEstablished
reportParingSuccess
updateMode
updateSpiBtState
enableHUBluetoothInterface
setBluetoothMACAddrs
```

The binary also identifies the Bluetooth controller source path as:

```text
.../mhd/dio_manager/dio_manager_app/src/controller/asi/BluetoothController.cxx
```

This establishes a DIO-side Bluetooth smartphone integration boundary with explicit CarPlay state handling.

It does **not** establish the missing Bluetooth iAP2 → DIO packet/session handoff.

## DIO environment-setting lead

The MU0678 `dio_manager` binary contains:

```text
uap0
carplay0
MDNS_DIRECTLINK_IFACE=carplay0
setEnv
getEnvName
putenv(%s): %s
startMdnsdProcess
restartMdnsd
stopMdnsdProcess
```

This narrows the producer search for the mDNS direct-link environment.

The evidence is deliberately classified as a lead only. The recovered occurrence of `MDNS_DIRECTLINK_IFACE=carplay0` is embedded in DIO configuration/persistence metadata, and the exact instruction-level edge proving:

```text
setEnv()
    ↓
putenv("MDNS_DIRECTLINK_IFACE=carplay0")
    ↓
startMdnsdProcess()
```

has not been recovered.

Therefore E-043 does not upgrade E-031.

## Current boundary map

```text
Bluetooth
   │
   ▼
Bluetooth smartphone integration
   │
   ├── iAP2-capable transport machinery
   │        │
   │        ├── Bluetooth control/update messages
   │        └── Wi-Fi configuration messages
   │
   └── DIO CarPlay state
            │
            └── AirPlay / mDNS
```

This is a capability/control-plane map, not a proven end-to-end execution path.

## Do not infer

- Bluetooth feature-startup code proves production Wireless CarPlay is enabled.
- `RequestAccessoryWiFiConfigurationInformation` proves a live iPhone Wi-Fi provisioning exchange.
- DIO `BluetoothSmartphoneIntegration` proves Bluetooth iAP2 packets already reach DIO.
- `setEnv` / `putenv` strings prove DIO exports `MDNS_DIRECTLINK_IFACE` to the running daemon.
- Presence of both `uap0` and `carplay0` proves either interface is selected for a wireless CarPlay session.

## Completion criterion

Complete only when the production activation branch, concrete transport object, Bluetooth/Wi-Fi iAP2 session, DIO handoff and resulting AirPlay session are correlated in one reproducible chain.
