# MHI2 Wireless Capability Breakdown

This document records the reverse-engineered wireless/hotspot architecture of the MHI2 system. It is intentionally evidence-driven: conclusions are based on firmware contents, configuration, binaries and runtime behaviour. Unproven areas are explicitly identified rather than inferred.

---

# 1. Current Mapped Architecture

This is the architecture currently established from the available MHI2 evidence.

```mermaid
flowchart TB
    HMI["MMI / HMI"]

    PROXY["ASI / DSI proxies"]

    LAUNCHER["connectivity_launcher"]

    subgraph SERVICES["Connectivity Services"]
        CM["connectionmanager"]
        TEL["telephone"]
        BT["bluetooth"]
        BTSTACK["btstack"]
        UPNP["dev-upnp"]
        NAD["nad"]
    end

    CFG["/tmp/uaputl.cfg"]
    UAPUTL["uaputl"]

    subgraph WLAN["Marvell 8787 WLAN"]
        FW["WLAN / BT firmware"]
        DRIVER["devnp-mrvl_wlan-sdiorm"]
        UAP["uap0<br/>10.173.189.1/24"]
        MLAN["mlan0"]
    end

    WPA["wpa_supplicant"]
    CLIENTS["Wi-Fi clients"]

    DNSMASQ["dnsmasq<br/>DHCP + DNS"]
    PF["PF<br/>NAT / routing / isolation"]

    subgraph WAN["External Data Network"]
        PPP["ppp0"]
        ECM["ecm0"]
        NADDEV["/dev/NAD/AT_port"]
        MODEM["Cellular modem"]
        INTERNET["Internet"]
    end

    HMI --> PROXY
    PROXY --> LAUNCHER

    LAUNCHER --> CM
    LAUNCHER --> TEL
    LAUNCHER --> BT
    LAUNCHER --> BTSTACK
    LAUNCHER --> UPNP
    LAUNCHER --> NAD

    CM --> CFG
    CFG --> UAPUTL
    UAPUTL --> FW

    FW --> DRIVER
    DRIVER --> UAP
    DRIVER --> MLAN

    MLAN --> WPA

    UAP --> CLIENTS

    CLIENTS --> DNSMASQ
    CLIENTS --> PF

    PF --> PPP
    PF --> ECM

    NAD --> NADDEV
    NADDEV --> MODEM

    PPP --> MODEM
    ECM --> MODEM
    MODEM --> INTERNET
```

## Current Proven Hotspot Flow

```mermaid
flowchart LR
    HMI["MMI / HMI"]
    CM["connectionmanager"]
    CFG["/tmp/uaputl.cfg"]
    UAPUTL["uaputl"]
    FW["Marvell 8787"]
    UAP["uap0"]
    CLIENT["Wi-Fi client"]
    DNS["dnsmasq"]
    PF["PF"]
    WAN["ppp0 / ecm0"]
    NAD["NAD / cellular"]

    HMI --> CM
    CM --> CFG
    CFG --> UAPUTL
    UAPUTL --> FW
    FW --> UAP
    UAP --> CLIENT

    CLIENT --> DNS
    CLIENT --> PF
    PF --> WAN
    WAN --> NAD
```

The critical distinction is:

- `uap0` is the production hotspot/AP interface.
- `mlan0` is the Wi-Fi client/station interface.
- `uaputl` controls the Marvell AP/WLAN firmware.
- `dnsmasq` provides DHCP and DNS.
- PF provides routing, NAT and isolation.
- `ppp0` / `ecm0` provide the external data path.

---

# 2. What Remains to Be Traced

The architecture above is the current map. The following are the remaining gaps.

## 2.1 Runtime Generation of `/tmp/uaputl.cfg`

We know:

```mermaid
flowchart LR
    CM["connectionmanager"]
    CFG["/tmp/uaputl.cfg"]
    UAPUTL["uaputl"]
    FW["Marvell firmware"]

    CM --> CFG
    CFG --> UAPUTL
    UAPUTL --> FW
```

Still unknown:

- who creates `/tmp/uaputl.cfg`
- which function writes it
- when it is generated
- how the SSID is generated
- how WPA credentials are generated
- which values come from persistent configuration
- which values are generated at runtime

---

## 2.2 Exact Production `uaputl` Sequence

`start-ap.sh` proves that `uaputl` performs low-level AP configuration.

The remaining task is to trace the complete production sequence:

```mermaid
flowchart LR
    START["Hotspot startup"]
    CFG["/tmp/uaputl.cfg"]
    SCRIPT["start-ap.sh"]
    UAPUTL["uaputl"]
    FW["Marvell firmware"]
    UAP["uap0"]

    START --> CFG
    START --> SCRIPT
    CFG --> SCRIPT
    SCRIPT --> UAPUTL
    UAPUTL --> FW
    FW --> UAP
```

---

## 2.3 Actual Production WAN Bearer

The framework supports:

```text
ECM
PPP
RMNET
```

The remaining task is to establish which path is actually selected on the target production configuration.

```mermaid
flowchart TB
    NAD["NAD"]

    ECM["ECM"]
    PPP["PPP"]
    RMNET["RMNET"]

    NAD --> ECM
    NAD --> PPP
    NAD --> RMNET
```

We should not infer the final bearer solely from the presence of these components.

---

## 2.4 HMI → Connectivity Runtime Path

The architecture establishes the proxy boundary:

```mermaid
flowchart LR
    HMI["WLAN / Connectivity HMI"]
    PROXY["ASI / DSI"]
    FRAMEWORK["Connectivity framework"]
    CM["connectionmanager"]

    HMI --> PROXY
    PROXY --> FRAMEWORK
    FRAMEWORK --> CM
```

The remaining work is method-level tracing of the actual calls and events.

---

## 2.5 WLAN Service Ownership

PF exposes numerous WLAN service ports.

The remaining work is to associate each port with its actual owning process/service.

```mermaid
flowchart LR
    WLAN["WLAN"]
    PORTS["WLAN service ports"]
    SERVICES["Partially mapped service owners"]

    WLAN --> PORTS
    PORTS --> SERVICES
```

---

# 3. Component / Subsystem Breakdown

## 3.1 MMI / HMI

The HMI contains distinct WLAN and connectivity layers:

```text
hmi_App_Wlan_Main
hmi_App_Wlan_DSI
hmi_App_Wlan_HMI

hmi_App_Connectivity_Main
hmi_App_Data_Main
hmi_App_Umts_Main
```

The corresponding WLAN proxy infrastructure includes:

```text
libasimmxwlanproxy.so
libasimmxDiagWlanTypesproxy.so
```

```mermaid
flowchart TB
    HMI["MMI / HMI"]

    WLAN["WLAN HMI"]
    CONNECTIVITY["Connectivity HMI"]
    DATA["Data HMI"]
    UMTS["UMTS HMI"]

    PROXY["ASI / DSI"]
    FRAMEWORK["Connectivity framework"]

    HMI --> WLAN
    HMI --> CONNECTIVITY
    HMI --> DATA
    HMI --> UMTS

    WLAN --> PROXY
    CONNECTIVITY --> PROXY
    PROXY --> FRAMEWORK
```

---

## 3.2 `connectivity_launcher`

`connectivity_launcher` is the production connectivity supervisor.

It launches:

```text
telephone
bluetooth
connectionmanager
messaging
dev-upnp
btstack
nad
```

```mermaid
flowchart TB
    LAUNCHER["connectivity_launcher"]

    LAUNCHER --> TELEPHONE["telephone"]
    LAUNCHER --> BLUETOOTH["bluetooth"]
    LAUNCHER --> CM["connectionmanager"]
    LAUNCHER --> MESSAGING["messaging"]
    LAUNCHER --> UPNP["dev-upnp"]
    LAUNCHER --> BTSTACK["btstack"]
    LAUNCHER --> NAD["nad"]
```

`connectionmanager` has a startup dependency on:

```text
/tmp/uap0
```

Therefore:

```mermaid
flowchart LR
    WLAN["WLAN initialization"]
    UAP0["/tmp/uap0"]
    CM["connectionmanager"]

    WLAN --> UAP0
    UAP0 --> CM
```

---

## 3.3 Marvell 8787

The MHI2 wireless controller is a Marvell 8787.

Firmware:

```text
sd8787_uapsta.bin
w8787_wlan_SDIO_bt_SDIO.bin
```

Host-side components:

```text
io-sdiorm-mib2
devnp-mrvl_wlan-sdiorm.so
mvload
uaputl
```

```mermaid
flowchart TB
    QNX["QNX"]

    SDIO["io-sdiorm-mib2"]
    DRIVER["devnp-mrvl_wlan-sdiorm.so"]
    MVLOAD["mvload"]

    MARVELL["Marvell 8787"]

    WLANFW["WLAN firmware"]
    BTFW["Bluetooth firmware"]

    QNX --> SDIO
    QNX --> DRIVER
    QNX --> MVLOAD

    SDIO --> MARVELL
    DRIVER --> MARVELL
    MVLOAD --> MARVELL

    MARVELL --> WLANFW
    MARVELL --> BTFW
```

---

## 3.4 `uap0`

`uap0` is the production hotspot/AP interface.

Startup:

```text
if_up -p -r 100 uap0
ifconfig uap0 up
ifconfig uap0 mediaopt hostap
ifconfig uap0 10.173.189.1 netmask 255.255.255.0
```

```mermaid
flowchart LR
    MARVELL["Marvell 8787"]
    UAP["uap0"]
    IP["10.173.189.1/24"]
    CLIENTS["Wi-Fi clients"]

    MARVELL --> UAP
    UAP --> IP
    UAP --> CLIENTS
```

---

## 3.5 `mlan0`

`mlan0` is the station/client interface.

It is started with:

```text
wpa_supplicant -Bi mlan0
```

```mermaid
flowchart LR
    MARVELL["Marvell 8787"]
    MLAN["mlan0"]
    WPA["wpa_supplicant"]
    NETWORK["External Wi-Fi"]

    MARVELL --> MLAN
    MLAN --> WPA
    WPA --> NETWORK
```

This is separate from the hotspot/AP path.

---

## 3.6 `connectionmanager`

Production configuration contains:

```text
autoProfilePath
customerProfilePath
uaputlCfgPath = /tmp/uaputl.cfg
onlineIdleTimeInS = 300
```

```mermaid
flowchart TB
    CM["connectionmanager"]

    AUTO["/eso/telephone/autoconf.xml"]
    CUSTOMER["/eso/bin/customer_apn/autoconf.xml"]
    CFG["/tmp/uaputl.cfg"]

    UAPUTL["uaputl"]
    FW["Marvell firmware"]

    CM --> AUTO
    CM --> CUSTOMER
    CM --> CFG
    CFG --> UAPUTL
    UAPUTL --> FW
```

---

## 3.7 `uaputl`

`uaputl` is the low-level control interface used to configure the Marvell WLAN firmware.

`start-ap.sh` uses commands including:

```text
uaputl bss_stop
uaputl sys_config
uaputl hostcmd
uaputl deepsleep
uaputl coex_config
uaputl aggrpriotbl
```

```mermaid
flowchart LR
    CONFIG["Runtime AP configuration"]
    UAPUTL["uaputl"]
    FW["Marvell WLAN firmware"]

    CONFIG --> UAPUTL
    UAPUTL --> FW
```

---

## 3.8 `dnsmasq`

`dnsmasq` provides both DHCP and DNS for the hotspot.

```mermaid
flowchart TB
    CLIENT["Wi-Fi clients"]
    UAP["uap0"]
    DNSMASQ["dnsmasq"]

    DHCP["DHCP"]
    DNS["DNS"]

    CLIENT --> UAP
    UAP --> DNSMASQ

    DNSMASQ --> DHCP
    DNSMASQ --> DNS
```

DHCP range:

```text
10.173.189.10 - 10.173.189.99
```

Leases:

```text
/ramdisk/dnsmasq.leases
```

---

## 3.9 PF

PF performs more than NAT.

Its responsibilities include:

- NAT
- routing policy
- WLAN isolation
- external-interface filtering
- service-port filtering
- DNS server state
- traffic classification

```mermaid
flowchart TB
    WLAN["uap0 / WLAN"]

    PF["PF"]

    NAT["NAT"]
    ROUTING["Routing"]
    FILTER["Filtering / isolation"]
    SERVICES["WLAN services"]

    WLAN --> PF

    PF --> NAT
    PF --> ROUTING
    PF --> FILTER
    PF --> SERVICES
```

---

## 3.10 NAD / Cellular Data

The NAD abstraction sits between the connectivity framework and the modem.

```mermaid
flowchart LR
    CONNECTIVITY["Connectivity framework"]
    NAD["nad"]
    AT["/dev/NAD/AT_port"]
    MODEM["Cellular modem"]

    CONNECTIVITY --> NAD
    NAD --> AT
    AT --> MODEM
```

Possible data mechanisms:

```text
ECM
PPP
RMNET
```

---

## 3.11 PPP

The PPP path is:

```mermaid
flowchart LR
    MODEM["Modem"]
    PPPD["pppd"]
    PPP0["ppp0"]
    ROUTING["Routing"]
    DNS["DNS"]
    PF["PF"]

    MODEM --> PPPD
    PPPD --> PPP0
    PPP0 --> ROUTING
    PPP0 --> DNS
    DNS --> PF
```

`/etc/ppp/ip-up` establishes routing, DNS and PF state when PPP becomes active.

---

## 3.12 ECM

ECM provides another external network path:

```mermaid
flowchart LR
    MODEM["Modem / USB"]
    ECM["ECM"]
    ECM0["ecm0"]
    PF["PF"]
    WLAN["uap0"]

    MODEM --> ECM
    ECM --> ECM0
    ECM0 --> PF
    PF --> WLAN
```

ECM state changes can trigger route, DNS, PF and `dnsmasq` reconfiguration.

---

## 3.13 UPnP

UPnP is a first-class connectivity service.

```mermaid
flowchart LR
    LAUNCHER["connectivity_launcher"]
    UPNP["dev-upnp"]
    WLAN["WLAN"]
    MULTICAST["239.255.255.0/24"]
    PORTS["49152 / 49153–49162"]

    LAUNCHER --> UPNP
    UPNP --> WLAN
    WLAN --> MULTICAST
    WLAN --> PORTS
```

---

## 3.14 WLAN Service Surface

The WLAN exposes services including:

| Port | Function |
|---:|---|
| 21001 | WLAN tracing |
| 20001 | GEMIB |
| 30001 | SWDL fallback |
| 23100 | Video |
| 23101 | Video control |
| 22222–22223 | Audio |
| 21600–21605 | SSL / control |
| 25010 | EXLAP |
| 28500 | EXLAP UDP |
| 49152 | UPnP |
| 49153–49162 | UPnP |

These should be treated as a service surface to be mapped to their owning processes.

```mermaid
flowchart TB
    WLAN["WLAN"]

    TRACE["WLAN tracing"]
    GEMIB["GEMIB"]
    SWDL["SWDL"]
    VIDEO["Video"]
    AUDIO["Audio"]
    CONTROL["Control"]
    EXLAP["EXLAP"]
    UPNP["UPnP"]

    WLAN --> TRACE
    WLAN --> GEMIB
    WLAN --> SWDL
    WLAN --> VIDEO
    WLAN --> AUDIO
    WLAN --> CONTROL
    WLAN --> EXLAP
    WLAN --> UPNP
```

---

# 4. Evidence Status

## Proven

- `uap0` is the hotspot/AP interface.
- `mlan0` is the Wi-Fi client/station interface.
- `uap0` uses `10.173.189.1/24`.
- `uap0` uses `mediaopt hostap`.
- `uaputl` controls the Marvell WLAN firmware.
- `/tmp/uaputl.cfg` is the runtime AP configuration path.
- `dnsmasq` provides DHCP and DNS.
- PF provides NAT, routing and WLAN security policy.
- `ppp0` and `ecm0` are supported external data interfaces.
- Marvell 8787 is the WLAN controller.
- `io-sdiorm-mib2` provides the SDIO hardware layer.
- `devnp-mrvl_wlan-sdiorm.so` is the QNX WLAN driver.
- `connectivity_launcher` supervises the connectivity services.
- Persistent hotspot and WLAN-client configuration are separate.
- UPnP is integrated into the connectivity architecture.

## Partially Traced

- Exact runtime generation of `/tmp/uaputl.cfg`.
- Exact production `uaputl` startup sequence.
- Exact modem/data-bearer selection.
- Exact connectivity-manager state transitions.
- Exact HMI → connectivity method-level call paths.
- Ownership of individual WLAN service ports.

## Not Yet Proven

- Exact SSID generation function.
- Exact WPA credential generation function.
- Exact production modem bearer under every MU0678 configuration.
- Complete runtime call graph from HMI to Marvell firmware.
- Exact relationship between every WLAN service port and its owning process.
