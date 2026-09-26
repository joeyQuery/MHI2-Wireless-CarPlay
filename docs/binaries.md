# MHI2 Binary Inventory

Authoritative inventory of binaries, libraries and processes relevant to the Wireless CarPlay investigation.

> Presence in the firmware proves that a component exists in the image. It does **not** prove that it participates in the production Wireless CarPlay path.

## CarPlay / AirPlay

| Component | Known role | Trace status |
|---|---|---|
| `dio_manager` | MHI2 CarPlay/DIO integration boundary | Partial |
| `libairplay.so` | AirPlay receiver/session and Bonjour/mDNS functionality | Partial |
| `devnp-usbdnet.so` | USB network driver producing `carplay0` | Production path established |

Relevant DIO symbols:

```text
requestCarPlay
checkCarPlayCompatibility
notifyiAP2DeviceConnected
notifyiAP2DeviceDisconnected
iAP2Connect
onEvent_sessionCreated
onEvent_sessionFinalized
onEvent_sessionCurrentModeChanged
onEvent_sessionChangeModeCompletion

AirPlayReceiverServer
AirPlayReceiverSession
AirPlayReceiverSessionSetup
AirPlayReceiverSessionScreen
AirPlayReceiverSessionChangeModes
AirPlayReceiverSessionSetSecurityInfo
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
```

## Bluetooth

| Component | Known role | Trace status |
|---|---|---|
| `bluetooth` | High-level production Bluetooth service | Partial |
| `btstack` | Bluetooth protocol-stack process | Partial |
| `libasimmxconnectivity_bluetooth_iapproxy.so` | Bluetooth-side iAP integration candidate | Partial |

Relevant configuration:

```text
enableIap=false
topologyLogic=1
```

The meaning of `topologyLogic=1` and the exact `enableIap` control flow remain unresolved.

## iAP / iAP2

| Component | Known role | Trace status |
|---|---|---|
| `/eso/bin/apps/iap` | iAP application/process | Partial |
| `libiap2client.so` | iAP2 client component | Partial |
| `ipod-drvr-iap2.so` | iAP2/iPod-side component | Partial |
| `mss-ipodiap2.so` | iAP2-related component | Partial |
| `devu-iap2-tegra3-ci.so` | iAP2 device/transport component | Partial |
| `devu-iap2ncm-tegra3-ci.so` | iAP2 NCM-related component | Partial |
| `iap2cli` | iAP2 diagnostic/client utility | Partial |

Production CarPlay currently exposes the USB-derived `/dev/ipod0` boundary. The transport abstraction behind it remains unresolved.

## WLAN / Connectivity

| Component | Known role | Trace status |
|---|---|---|
| `io-sdiorm-mib2` | SDIO resource-manager layer | Partial |
| `devnp-mrvl_wlan-sdiorm.so` | Marvell WLAN network driver | Partial |
| `mvload` | Firmware loading support | Partial |
| `connectionmanager` | WLAN/AP orchestration | Partial |
| `uaputl` | Marvell AP firmware control | Established |
| `dnsmasq` | DHCP/DNS for AP environment | Established |

Relevant Marvell firmware:

```text
sd8787_uapsta.bin
w8787_wlan_SDIO_bt_SDIO.bin
```

## Connectivity Supervisor / Other Services

```text
connectivity_launcher
telephone
messaging
dev-upnp
nad
```

These are part of the broader connectivity architecture. Their presence does not make them Wireless CarPlay components by itself.

## Inventory Discipline

For every newly analysed binary, record:

1. exact path;
2. firmware/source baseline;
3. architecture;
4. process vs shared library vs driver vs firmware;
5. relevant symbols/strings;
6. known callers/consumers;
7. runtime evidence, if any;
8. current evidence classification;
9. unresolved questions.

Do not promote a component from “present” to “used by Wireless CarPlay” without a traceable relationship.
