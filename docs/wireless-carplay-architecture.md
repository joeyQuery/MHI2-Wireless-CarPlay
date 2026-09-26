# MHI2 Wireless CarPlay Architecture

Evidence-driven integration map for Wireless CarPlay on Audi MHI2 / MU0678-class QNX firmware.

> **Status:** Active reverse engineering / implementation architecture  
> **Purpose:** Connect the independently mapped Bluetooth, iAP2, WLAN, mDNS, AirPlay and CarPlay subsystems into one end-to-end trace.  
> **Evidence rule:** A component being present does not prove that it is used by the production Wireless CarPlay path. Proven facts, runtime observations and unresolved hypotheses are kept separate.

---

## Edge notation

Architecture diagrams use **solid arrows only for relationships established at the level claimed by the diagram** and **dashed arrows for target, inferred or unresolved relationships**. A subsystem-level relationship is not automatically a function-level execution edge. A dashed arrow is never evidence of a completed production path. Recovered execution/data-flow traces live in [`traces/`](../traces/README.md).

# 1. Current Mapped Architecture

## 1.1 System-level architecture

The production MHI2 image already contains the major pieces required by a Wireless CarPlay implementation.

~~~mermaid
flowchart TB
    PHONE["iPhone"]

    subgraph RADIO["Shared Marvell 8787"]
        BT_RADIO["Bluetooth"]
        WLAN_RADIO["WLAN"]
    end

    subgraph CONNECTIVITY["MHI2 Connectivity"]
        BLUETOOTH["bluetooth"]
        BTSTACK["btstack"]
        CM["connectionmanager"]
        UAPUTL["uaputl"]
    end

    subgraph NETWORK["MHI2 Network Services"]
        UAP["uap0"]
        DNS["dnsmasq"]
        MDNS["mDNS"]
        PF["PF"]
    end

    subgraph CARPLAY["CarPlay Application Layer"]
        DIO["dio_manager"]
        IAP2["iAP2 service"]
        AIRPLAY["libairplay.so"]
    end

    subgraph MEDIA["CarPlay Media / Control"]
        VIDEO["Screen / video"]
        AUDIO["Audio"]
        HID["HID / HMI"]
    end

    PHONE <-->|"Bluetooth"| BT_RADIO
    PHONE <-->|"Wi-Fi"| WLAN_RADIO
    BT_RADIO --> BTSTACK
    BTSTACK --> BLUETOOTH
    BLUETOOTH -.-> IAP2
    WLAN_RADIO --> UAP
    CM --> UAPUTL
    UAPUTL --> WLAN_RADIO
    UAP --> DNS
    UAP --> MDNS
    UAP --> PF
    IAP2 -.-> DIO
    MDNS -.-> DIO
    DIO --> AIRPLAY
    AIRPLAY --> VIDEO
    AIRPLAY --> AUDIO
    DIO --> HID
~~~

This is an architectural map. It does not claim every arrow is already proven at function-call level.

## 1.2 Production USB CarPlay path

The current production CarPlay transport is explicitly tied to USB infrastructure.

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    USB["USB"]
    NCM["USB NCM"]
    CARPLAY0["carplay0"]
    MDNS["MDNS_DIRECTLINK_IFACE=carplay0"]
    DIO["dio_manager"]
    IPOD["/dev/ipod0"]
    IAP2["iAP2 driver"]
    AIRPLAY["libairplay.so"]
    CP["CarPlay session"]

    PHONE --> USB
    USB --> NCM
    NCM --> CARPLAY0
    CARPLAY0 --> MDNS
    MDNS --> DIO
    PHONE --> IPOD
    IPOD --> IAP2
    IAP2 --> DIO
    DIO --> AIRPLAY
    AIRPLAY --> CP
~~~

Production configuration identifies `/dev/ipod0` as the iAP2 device. The USB networking path creates `carplay0` using `devnp-usbdnet.so`.

## 1.3 Wireless target architecture

The target is to replace transport-specific USB dependencies while retaining the existing WLAN, Bluetooth, DIO and AirPlay layers where their interfaces permit it.

~~~mermaid
flowchart LR
    PHONE["iPhone"]

    subgraph WIRELESS["Wireless Transport"]
        BT["Bluetooth"]
        WIFI["Wi-Fi / uap0"]
    end

    subgraph BOOTSTRAP["CarPlay Bootstrap"]
        IAP2["Wireless iAP2 / bootstrap"]
    end

    subgraph NETWORK["Wireless CarPlay Network"]
        DHCP["DHCP / DNS"]
        MDNS["mDNS / Bonjour"]
    end

    subgraph INTEGRATION["MHI2 CarPlay Integration"]
        DIO["dio_manager"]
        AIRPLAY["libairplay.so"]
    end

    subgraph SESSION["CarPlay Session"]
        SCREEN["Screen"]
        AUDIO["Audio"]
        HID["HID"]
    end

    PHONE <-->|"Bluetooth"| BT
    PHONE <-->|"Wi-Fi"| WIFI
    BT -.-> IAP2
    WIFI -.-> DHCP
    WIFI -.-> MDNS
    IAP2 -.-> DIO
    MDNS -.-> DIO
    DIO -.-> AIRPLAY
    AIRPLAY -.-> SCREEN
    AIRPLAY -.-> AUDIO
    DIO -.-> HID
~~~

The intended transformation is:

~~~text
CURRENT
iPhone
  ├── USB ──> carplay0 ──> DIO
  └── USB iAP2 ──> /dev/ipod0 ──> DIO
                              └──> libairplay

TARGET
iPhone
  ├── Bluetooth ──> wireless iAP2 ──┐
  └── Wi-Fi ──> uap0 ─> mDNS ───────┤
                                    ▼
                                   DIO
                                    │
                                    ▼
                               libairplay
~~~

## 1.4 WLAN substrate

~~~mermaid
flowchart TB
    START["MHI2 wireless startup"]
    SDIO["io-sdiorm-mib2"]
    DEV["/dev/sdio0"]
    MVLOAD["mvload"]
    FW["Marvell WLAN / BT firmware"]
    DRIVER["devnp-mrvl_wlan-sdiorm.so"]
    UAPUTL["uaputl"]
    CM["connectionmanager"]
    UAP["uap0"]
    DNS["dnsmasq"]
    PF["PF"]
    PHONE["iPhone"]

    START --> SDIO
    SDIO --> DEV
    DEV --> MVLOAD
    MVLOAD --> FW
    FW --> DRIVER
    DRIVER --> UAP
    CM --> UAPUTL
    UAPUTL --> FW
    UAP --> DNS
    UAP --> PF
    PHONE <-->|"Wi-Fi"| UAP
~~~

Known AP state includes `uap0` and `10.173.189.1/24`. DHCP/DNS and PF infrastructure already surround this interface.

## 1.5 Bluetooth / WLAN coexistence

~~~mermaid
flowchart TB
    SDIO["SDIO"]
    MARVELL["Marvell 8787"]
    WLAN["WLAN"]
    BT["Bluetooth"]
    COEX["WLAN / Bluetooth coexistence"]

    SDIO --> MARVELL
    MARVELL --> WLAN
    MARVELL --> BT
    WLAN <--> COEX
    BT <--> COEX
~~~

The production AP startup uses `coex_config`; the available configuration contains Bluetooth/WLAN scheduling parameters. This proves an existing coexistence mechanism, not a completed Wireless CarPlay path.

---

# 2. What Remains to Be Traced

## 2.1 Bluetooth → iAP2 bootstrap

Known:

~~~text
bluetooth
  │
  └── libasimmxconnectivity_bluetooth_iapproxy.so
~~~

The firmware also contains iAP/iAP2 components and the Bluetooth configuration explicitly contains:

~~~text
enableIap=false
~~~

Remaining:

1. Identify the exact code path gated by `enableIap`.
2. Determine how a Bluetooth CarPlay-capable phone reaches iAP2.
3. Determine whether an existing iAP2 transport abstraction can be activated without replacing DIO.
4. Trace the Bluetooth CarPlay state/type into the iAP2 connection request.
5. Establish the bootstrap transition into the Wi-Fi/AirPlay session.

## 2.2 DIO iAP2 transport boundary

Known production path:

~~~mermaid
flowchart LR
    DIO["dio_manager / CIpodAP2Service"]
    DEV["/dev/ipod0"]
    DRIVER["iAP2 driver"]
    DIO --> DEV --> DRIVER
~~~

The unresolved boundary is:

~~~mermaid
flowchart LR
    DIO["CIpodAP2Service"]
    CURRENT["/dev/ipod0"]
    ABSTRACTION["Unknown transport abstraction"]
    WIRELESS["Wireless iAP2"]

    DIO --> CURRENT
    DIO -.-> ABSTRACTION
    ABSTRACTION -.-> WIRELESS
~~~

The key question is whether the service is intrinsically USB-specific or whether its transport can be redirected.

## 2.3 mDNS direct-link interface

Known:

~~~text
MDNS_DIRECTLINK_IFACE=carplay0
~~~

Remaining trace:

~~~mermaid
flowchart LR
    ENV["MDNS_DIRECTLINK_IFACE"]
    CONSUMER["Unknown consumer"]
    SOCKET["mDNS / socket setup"]
    AIRPLAY["libairplay.so"]

    ENV -.-> CONSUMER
    CONSUMER -.-> SOCKET
    SOCKET -.-> AIRPLAY
~~~

The critical experiment is to locate the consumer and determine whether `uap0` is sufficient as the wireless CarPlay interface.

## 2.4 AirPlay interface selection

Known AirPlay symbols include:

~~~text
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr

SocketSetBoundInterface
SocketSetPacketReceiveInterface
SocketSetMulticastInterface

IsWiFiNetworkInterface
IsUSBNetworkInterface
~~~

Remaining trace:

~~~mermaid
sequenceDiagram
    participant D as dio_manager
    participant A as libairplay
    participant S as AirPlay Screen
    participant N as Network

    D->>A: create/configure session
    D->>S: transport/interface setup
    S->>A: SetIFName / SetTransportType / SetClientIfMACAddr
    A->>N: bind / multicast / receive setup
~~~

The caller, argument values, transport value, interface name, client MAC and call timing must be recovered.

## 2.5 End-to-end Wireless CarPlay trace

~~~mermaid
sequenceDiagram
    participant P as iPhone
    participant B as Bluetooth
    participant W as uap0
    participant I as iAP2
    participant D as dio_manager
    participant M as mDNS
    participant A as libairplay
    participant C as CarPlay

    P->>B: Bluetooth bootstrap
    B->>I: establish iAP2 path
    P->>W: Wi-Fi association
    W->>M: network available
    P->>M: discovery
    M->>D: discovery event
    I->>D: iAP2 event
    D->>A: create/configure session
    A->>C: CarPlay session
    P->>A: AirPlay setup/control
    A->>C: media/session data
~~~

Every untraced arrow remains an investigation target.

---

# 3. Trace Artifacts

The current transport-boundary trace set is:

- [TRACE-001 — Bluetooth → iAP2](../traces/TRACE-001-bluetooth-iap2.md)
- [TRACE-002 — DIO → iAP2](../traces/TRACE-002-dio-iap2.md)
- [TRACE-003 — MDNS_DIRECTLINK_IFACE](../traces/TRACE-003-mdns-interface.md)
- [TRACE-004 — DIO → AirPlay](../traces/TRACE-004-dio-airplay.md)
- [TRACE-005 — AirPlay network binding](../traces/TRACE-005-airplay-network.md)
- [TRACE-006 — session correlation](../traces/TRACE-006-session-correlation.md)

All six are intentionally marked Partial or Target. None claims the unresolved wireless path is implemented.

# 4. Component / Subsystem Breakdown

## 4.1 Marvell 8787 / SDIO

Relevant host components:

~~~text
io-sdiorm-mib2
devnp-mrvl_wlan-sdiorm.so
mvload
~~~

Relevant symbols include:

~~~text
mv8787_init
mv8787_start
mv8787_stop
mv8787_wlan_tx
mv8787_wlan_rx
mv8787_bt_rx
mv8787_intr_uap
mv8787_intr_bt
mv8787_start_uap
mv8787_uap_thread
mv8787_dnld_fw_w_hlp
~~~

~~~mermaid
flowchart TB
    QNX["QNX"]
    SDIORM["io-sdiorm-mib2"]
    SDIO["SDIO"]
    CHIP["Marvell 8787"]
    FW["WLAN / BT firmware"]
    WLAN["WLAN"]
    BT["Bluetooth"]

    QNX --> SDIORM
    SDIORM --> SDIO
    SDIO --> CHIP
    FW --> CHIP
    CHIP --> WLAN
    CHIP --> BT
~~~

## 4.2 connectionmanager

Relevant WLAN methods include:

~~~text
networking_4WLAN::activateAP
networking_4WLAN::setWLAN
networking_4WLAN::connectToApSync
networking_4WLAN::disconnectFromApSync
networking_4WLAN::newScanResultsSync
networking_4WLAN::checkForWlanClients
networking_4WLAN::setDefaultPassword
networking_4WLAN::setDefaultChannel
networking_4WLAN::setTxPower
networking_4WLAN::edMacEnable
networking_4WLAN::setPMode

execSyncUapUtl
generateDefaultPassword
generateRandomPassword
writeWpaConf
computePSK
~~~

~~~mermaid
flowchart LR
    CM["connectionmanager"]
    CFG["/tmp/uaputl.cfg"]
    SCRIPT["start-ap.sh"]
    UAPUTL["uaputl"]
    FW["Marvell firmware"]

    CM --> CFG
    CFG --> SCRIPT
    SCRIPT --> UAPUTL
    UAPUTL --> FW
~~~

## 4.3 uap0 / network services

~~~mermaid
flowchart TB
    PHONE["iPhone"]
    UAP["uap0
10.173.189.1/24"]
    DNSMASQ["dnsmasq
DHCP + DNS"]
    MDNS["mDNS"]
    PF["PF
filtering / NAT / queues"]
    WAN["ppp0 / ecm0"]
    NAD["NAD / modem"]

    PHONE <--> UAP
    UAP --> DNSMASQ
    UAP --> MDNS
    UAP --> PF
    PF --> WAN
    WAN --> NAD
~~~

Known DHCP range:

~~~text
10.173.189.10 - 10.173.189.99
~~~

## 4.4 bluetooth

Production configuration includes:

~~~text
enableIap=false
~~~

The firmware contains:

~~~text
libasimmxconnectivity_bluetooth_iapproxy.so
~~~

and integration types including:

~~~text
BluetoothSmartphoneIntegration
CARPLAY
ANDROID_AUTO
MIRRORLINK
NOTHING
~~~

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    BT["Bluetooth"]
    INTEGRATION["BluetoothSmartphoneIntegration"]
    CARPLAY["CARPLAY"]
    IAP["iAP / iAP2"]

    PHONE --> BT
    BT --> INTEGRATION
    INTEGRATION --> CARPLAY
    CARPLAY --> IAP
~~~

The exact call graph remains unresolved.

## 4.5 btstack

Relevant configuration/trace facilities include:

~~~text
hciStartupCapture=true
hciCaptureMaskedL2capChannels=false
stayMaster=true
wbsSupported=true
didSupported=true
clockOffsetUpdate=true

CON_BTSTACK
CON_BTSTACK_IA
CON_BTSTACK_IA_SAP
CON_BTSTACK_HCICAPTURE
~~~

These make HCI/L2CAP capture a high-value source for tracing the wireless bootstrap.

## 4.6 iAP / iAP2

Relevant components:

~~~text
/eso/bin/apps/iap
libasimmxconnectivity_bluetooth_iapproxy.so
devu-iap2-tegra3-ci.so
devu-iap2ncm-tegra3-ci.so
ipod-drvr-iap2.so
mss-ipodiap2.so
libiap2client.so
iap2cli
~~~

Production USB path:

~~~mermaid
flowchart TB
    PHONE["iPhone"]
    USB["USB"]
    IPOD["/dev/ipod0"]
    DRIVER["iAP2 driver"]
    DIO["dio_manager"]

    PHONE --> USB
    USB --> IPOD
    IPOD --> DRIVER
    DRIVER --> DIO
~~~

The key unresolved question is whether existing iAP2 components expose a reusable wireless transport boundary.

## 4.7 carplay0 / USB networking

The production USB network interface uses:

~~~text
devnp-usbdnet.so
name=carplay
ifconfig carplay0 up
~~~

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    USB["USB"]
    USBDNET["devnp-usbdnet.so"]
    CARPLAY0["carplay0"]
    DIO["dio_manager"]

    PHONE <--> USB
    USB --> USBDNET
    USBDNET --> CARPLAY0
    CARPLAY0 --> DIO
~~~

This is the network-side component Wireless CarPlay must no longer depend on.

## 4.8 DIO Manager

DIO is the central application integration boundary.

Relevant AirPlay symbols:

~~~text
AirPlayReceiverSession
AirPlayReceiverServer
AirPlayReceiverSessionSetup
AirPlayReceiverSessionScreen
AirPlayReceiverSessionChangeModes
AirPlayReceiverSessionSetSecurityInfo
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
~~~

Relevant CarPlay/session symbols:

~~~text
requestCarPlay
checkCarPlayCompatibility
onEvent_sessionCreated
onEvent_sessionFinalized
onEvent_sessionCurrentModeChanged
onEvent_sessionChangeModeCompletion
notifyiAP2DeviceConnected
notifyiAP2DeviceDisconnected
iAP2Connect
~~~

~~~mermaid
flowchart TB
    SMARTPHONE["smartphone_integrator"]
    DIO["dio_manager"]
    IAP["iAP2 service"]
    MDNS["mDNS"]
    AIRPLAY["libairplay.so"]
    SESSION["CarPlay session"]

    SMARTPHONE --> DIO
    DIO --> IAP
    DIO --> MDNS
    DIO --> AIRPLAY
    IAP --> SESSION
    AIRPLAY --> SESSION
~~~

## 4.9 mDNS / AirPlay

AirPlay contains:

~~~text
DNSServiceRegister
DNSServiceUpdateRecord
DNSServiceGetAddrInfo
DNSServiceQueryRecord

_airplay._tcp.

Registering Bonjour %s port %d
Registered Bonjour %s.%s%s
Updated Bonjour TXT
~~~

Interface helpers include:

~~~text
SocketSetBoundInterface
SocketSetPacketReceiveInterface
SocketSetMulticastInterface
IsWiFiNetworkInterface
IsUSBNetworkInterface
~~~

~~~mermaid
flowchart LR
    AIRPLAY["libairplay.so"]
    MDNS["Bonjour / mDNS"]
    IFACE["Interface selection"]
    WIFI["Wi-Fi"]
    USB["USB"]

    AIRPLAY --> MDNS
    AIRPLAY --> IFACE
    IFACE --> WIFI
    IFACE --> USB
~~~

The existence of Wi-Fi-aware helpers is evidence that the library has interface-selection machinery; it does not prove MHI2 currently invokes it with `uap0`.

## 4.10 AirPlay session media

~~~mermaid
flowchart TB
    SERVER["AirPlayReceiverServer"]
    SESSION["AirPlayReceiverSession"]
    SETUP["SETUP"]
    SCREEN["Screen"]
    AUDIO["Audio"]
    NTP["NTP"]
    HID["HID / control"]

    SERVER --> SESSION
    SESSION --> SETUP
    SESSION --> SCREEN
    SESSION --> AUDIO
    SESSION --> NTP
    SESSION --> HID
~~~

The implementation goal is to reach this existing session machinery through wireless transport rather than build a parallel receiver.

---

# 5. Evidence Status

## 5.1 Established

| Finding | Status |
|---|---|
| Marvell 8787 WLAN/BT hardware | **Proven** |
| SDIO driver path | **Proven** |
| `uap0` AP interface | **Proven** |
| `connectionmanager` WLAN control | **Proven** |
| `uaputl` Marvell AP control | **Proven** |
| DHCP/DNS infrastructure | **Proven** |
| PF infrastructure | **Proven** |
| Bluetooth service | **Proven** |
| Separate `btstack` executable | **Proven** |
| iAP/iAP2 components | **Proven** |
| Production `enableIap=false` | **Proven** |
| Production `/dev/ipod0` iAP2 configuration | **Proven** |
| USB `carplay0` interface | **Proven** |
| `MDNS_DIRECTLINK_IFACE=carplay0` | **Proven** |
| AirPlay receiver symbols | **Proven** |
| AirPlay Bonjour symbols | **Proven** |
| AirPlay Wi-Fi/USB interface helpers | **Proven** |
| DIO CarPlay state-machine symbols | **Proven** |
| DIO iAP2 connection symbols | **Proven** |
| Bluetooth/WLAN coexistence configuration | **Proven** |

## 5.2 Runtime evidence

The research establishes runtime evidence for:

- the WLAN/AP interface existing independently of USB CarPlay;
- CarPlay using a dedicated USB networking interface;
- DIO maintaining explicit iAP2 and CarPlay session state;
- AirPlay exposing interface/transport selection machinery;
- the MHI2 wireless subsystem sharing the Marvell 8787 between Bluetooth and WLAN.

Static evidence and runtime observations should remain separately labelled.

## 5.3 Not yet proven

Do **not** document these as working capabilities yet:

~~~text
Bluetooth -> wireless iAP2 -> DIO
uap0 -> CarPlay mDNS discovery
MDNS_DIRECTLINK_IFACE=uap0 is sufficient
DIO can replace /dev/ipod0 without modification
AirPlay SetTransportType accepts the required wireless value
DIO already contains a complete Wireless CarPlay path
A stock iPhone can complete Wireless CarPlay against the modified path
~~~

## 5.4 Evidence classes

Every new finding should be tagged as one of:

- **Static evidence** — strings, symbols, configuration or disassembly.
- **Runtime evidence** — observed process/interface/session behaviour.
- **Protocol evidence** — HCI, iAP2, mDNS or AirPlay traces.
- **Controlled experiment** — deliberate modification followed by observation.
- **Inference** — plausible interpretation requiring confirmation.

---

# 6. End State / Trace Objective

## 6.1 Final target

~~~mermaid
flowchart TB
    PHONE["iPhone"]

    subgraph TRANSPORT["Wireless Transport"]
        BT["Bluetooth"]
        WIFI["Wi-Fi / uap0"]
    end

    subgraph EXISTING["Existing MHI2 CarPlay Stack"]
        IAP["iAP2"]
        DIO["dio_manager"]
        MDNS["mDNS / Bonjour"]
        AIRPLAY["libairplay.so"]
        CP["CarPlay session"]
    end

    subgraph OUTPUT["Existing Media / HMI"]
        VIDEO["Video / NVSS"]
        AUDIO["Audio"]
        HID["HID / HMI"]
    end

    PHONE <-->|"Bootstrap"| BT
    PHONE <-->|"CarPlay network"| WIFI
    BT -.-> IAP
    WIFI -.-> MDNS
    IAP -.-> DIO
    MDNS -.-> DIO
    DIO -.-> AIRPLAY
    AIRPLAY -.-> CP
    CP -.-> VIDEO
    CP -.-> AUDIO
    CP -.-> HID
~~~

The desired implementation preserves the existing:

~~~text
Marvell 8787
uap0
dnsmasq
PF
mDNS
bluetooth
btstack
DIO
libairplay
NVSS
HMI
~~~

where their interfaces remain suitable.

The transport-specific USB dependencies to resolve are:

~~~text
/dev/ipod0
carplay0
MDNS_DIRECTLINK_IFACE=carplay0
~~~

## 6.2 Trace completion criteria

~~~mermaid
flowchart LR
    A["1. Bluetooth bootstrap"]
    B["2. Wireless iAP2"]
    C["3. Wi-Fi association"]
    D["4. mDNS discovery"]
    E["5. DIO CarPlay request"]
    F["6. AirPlay SETUP"]
    G["7. Screen stream"]
    H["8. Audio stream"]
    I["9. HID"]
    J["10. Reconnect"]

    A --> B --> C --> D --> E --> F --> G --> H --> I --> J
~~~

Evidence to capture:

~~~text
Bluetooth:
    HCI / btstack trace

iAP2:
    transport + protocol trace

Wi-Fi:
    association + IP configuration

mDNS:
    query / response / Bonjour registration

DIO:
    session and iAP2 events

AirPlay:
    SETUP / session / interface selection

Media:
    screen/audio packet flow

Control:
    HID / DSI behaviour

Recovery:
    disconnect / reconnect transitions
~~~

## 6.3 Final integration boundary

~~~mermaid
flowchart TB
    PHONE["iPhone"]

    BT["Bluetooth"]
    WIFI["Wi-Fi / uap0"]
    IAP["Wireless iAP2"]
    MDNS["mDNS"]
    DIO["DIO / CarPlay integration"]
    AIRPLAY["libairplay"]
    SESSION["CarPlay session"]

    VIDEO["Video"]
    AUDIO["Audio"]
    HID["HID"]

    PHONE --> BT
    PHONE --> WIFI
    BT --> IAP
    WIFI --> MDNS
    IAP --> DIO
    MDNS --> DIO
    DIO --> AIRPLAY
    AIRPLAY --> SESSION
    SESSION --> VIDEO
    SESSION --> AUDIO
    SESSION --> HID
~~~

The critical reverse-engineering questions are:

1. How is Wireless CarPlay iAP2 bootstrapped over Bluetooth?
2. What transport abstraction does DIO use for iAP2?
3. Where is `carplay0` selected as the CarPlay network interface?
4. How does `MDNS_DIRECTLINK_IFACE` enter the AirPlay/mDNS path?
5. What exact values are supplied to the AirPlay interface/transport APIs?
6. Can those boundaries be redirected to `uap0` and wireless iAP2 without replacing the CarPlay application layer?

---

# 7. Binary Reference Map

| Layer | Binary / component | Role |
|---|---|---|
| Connectivity supervisor | `/eso/bin/apps/connectivity_launcher` | Starts connectivity services |
| WLAN orchestration | `/eso/bin/apps/connectionmanager` | WLAN/AP configuration |
| AP control | `/armle/sbin/uaputl` | Marvell uAP control |
| SDIO hardware | `/armle/sbin/io-sdiorm-mib2` | Marvell 8787 host driver |
| WLAN network driver | `devnp-mrvl_wlan-sdiorm.so` | QNX WLAN interface |
| Bluetooth service | `/eso/bin/apps/bluetooth` | MHI2 Bluetooth integration |
| Bluetooth protocol | `/eso/bin/apps/btstack` | HCI/L2CAP/profile layer |
| iAP proxy | `libasimmxconnectivity_bluetooth_iapproxy.so` | Bluetooth-side iAP integration |
| iAP2 | `libiap2client.so` and related components | iAP2 infrastructure |
| USB CarPlay network | `devnp-usbdnet.so` | Creates `carplay0` |
| CarPlay integration | `dio_manager` | iAP2 + mDNS + AirPlay/session integration |
| AirPlay | `libairplay.so` | AirPlay receiver/session/media layer |
| DHCP/DNS | `dnsmasq` | Wireless client network services |
| Firewall/network | `PF` | Filtering/NAT/queues |
| mDNS | `mdnsd` / AirPlay Bonjour APIs | Discovery/service advertisement |

---

# 8. Immediate Reverse-Engineering Targets

## Target A — `enableIap`

Find the exact branch/function in `bluetooth` controlled by:

~~~text
enableIap=false
~~~

Determine exactly what enabling it changes.

## Target B — DIO iAP2 construction

Trace:

~~~text
CIpodAP2Service
    ->
iap2_connect
    ->
/dev/ipod0
~~~

Determine whether the device path is hard-coded or supplied through a configuration/constructor boundary.

## Target C — `MDNS_DIRECTLINK_IFACE`

Find every reference to:

~~~text
MDNS_DIRECTLINK_IFACE
~~~

and trace its value into socket/interface selection.

## Target D — AirPlay interface setup

Trace callers of:

~~~text
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
~~~

Record:

~~~text
caller
arguments
transport value
interface name
client MAC
call timing
~~~

## Target E — Wireless bootstrap

Use the existing Bluetooth HCI capture facilities to determine exactly what happens when an iPhone establishes the CarPlay-capable Bluetooth relationship.

## Target F — End-to-end correlation

Correlate:

~~~text
Bluetooth HCI
     +
iAP2
     +
Wi-Fi
     +
mDNS
     +
DIO slog
     +
AirPlay SETUP
~~~

into one timestamped session trace.

---

# 9. Architectural Conclusion

The evidence currently supports this model:

~~~mermaid
flowchart LR
    subgraph HARDWARE["Existing Hardware"]
        CHIP["Marvell 8787"]
    end

    subgraph TRANSPORT["Existing Wireless Transports"]
        BT["Bluetooth"]
        WLAN["WLAN / uap0"]
    end

    subgraph CP["Existing CarPlay Integration"]
        IAP["iAP2"]
        DIO["DIO"]
        AIRPLAY["AirPlay"]
    end

    CHIP --> BT
    CHIP --> WLAN
    BT --> IAP
    WLAN --> AIRPLAY
    IAP --> DIO
    AIRPLAY --> DIO
~~~

The production implementation is transport-bound to USB at specific integration points, especially:

~~~text
/dev/ipod0
carplay0
MDNS_DIRECTLINK_IFACE=carplay0
~~~

The Wireless CarPlay implementation should therefore be treated as a **transport adaptation of the existing MHI2 CarPlay stack**, not as a replacement WLAN or AirPlay implementation.

The architecture is fully traced only when the remaining transport boundaries are demonstrated at binary, runtime and protocol level.
