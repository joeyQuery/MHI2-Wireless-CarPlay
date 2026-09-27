# MHI2 Wi-Fi Architecture

Evidence-based reverse engineering of the MHI2 / MU0678-class Wi-Fi subsystem, focused on the existing WLAN infrastructure required for Wireless CarPlay.

> **Status:** Active research  
> **Platform:** Audi MHI2 / QNX  
> **Target:** Wireless CarPlay

---

# 1. Current Mapped Architecture

## 1.1 End-to-end WLAN architecture

~~~mermaid
flowchart TB
    PHONE["iPhone / WLAN client"]
    CHIP["Marvell 8787"]
    FW["Marvell WLAN / BT firmware"]
    SDIO["io-sdiorm-mib2"]
    DRIVER["devnp-mrvl_wlan-sdiorm.so"]
    CM["connectionmanager"]
    UAPUTL["uaputl"]
    UAP["uap0"]
    DNS["dnsmasq"]
    MDNS["mDNS / Bonjour"]
    PF["PF"]

    PHONE <--> UAP
    UAP --> DRIVER
    DRIVER --> CHIP
    CHIP --> FW
    CHIP --> SDIO
    SDIO --> DRIVER
    CM --> UAPUTL
    UAPUTL --> FW
    UAP --> DNS
    UAP --> MDNS
    UAP --> PF
~~~

The important point for Wireless CarPlay is that uap0 is an existing IP network interface independent of the USB CarPlay interface carplay0.

## 1.2 Shared WLAN / Bluetooth hardware

~~~mermaid
flowchart TB
    QNX["QNX"]
    SDIORM["io-sdiorm-mib2"]
    SDIO["SDIO"]
    MARVELL["Marvell 8787"]
    FW["WLAN / BT firmware"]
    WLAN["WLAN"]
    BT["Bluetooth"]

    QNX --> SDIORM
    SDIORM --> SDIO
    SDIO --> MARVELL
    FW --> MARVELL
    MARVELL --> WLAN
    MARVELL --> BT
~~~

Known firmware includes:

~~~text
sd8787_uapsta.bin
w8787_wlan_SDIO_bt_SDIO.bin
~~~

## 1.3 Production AP interface

Known:

~~~text
uap0
10.173.189.1/24
~~~

Known DHCP range:

~~~text
10.173.189.10 - 10.173.189.99
~~~

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    UAP["uap0"]
    DHCP["dnsmasq"]
    MDNS["mDNS"]
    PF["PF"]

    PHONE <--> UAP
    UAP --> DHCP
    UAP --> MDNS
    UAP --> PF
~~~

This establishes an existing MHI2 IP network suitable for investigating Wireless CarPlay. It does not prove that production CarPlay currently binds to it.

## 1.4 AP configuration path

~~~mermaid
flowchart LR
    CM["connectionmanager"]
    CFG["/tmp/uaputl.cfg"]
    UAPUTL["uaputl"]
    FW["Marvell AP firmware"]
    UAP["uap0"]

    CM --> CFG
    CFG --> UAPUTL
    UAPUTL --> FW
    FW --> UAP
~~~

---

# 2. What Remains to Be Traced

## 2.1 WLAN startup

~~~mermaid
flowchart LR
    START["Wireless startup"]
    SDIO["io-sdiorm-mib2"]
    DEV["/dev/sdio0"]
    MVLOAD["mvload"]
    FW["Marvell firmware"]
    DRIVER["WLAN driver"]
    IFACE["uap0"]

    START --> SDIO
    SDIO --> DEV
    DEV --> MVLOAD
    MVLOAD --> FW
    FW --> DRIVER
    DRIVER --> IFACE
~~~

Remaining:

- exact startup ordering;
- firmware load arguments;
- driver registration;
- interface creation timing;
- interaction with connectionmanager.

## 2.2 AP orchestration

Relevant connectionmanager methods include:

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
writeWpaConf
computePSK
generateDefaultPassword
generateRandomPassword
~~~

The exact runtime call order remains to be traced.

## 2.3 uaputl configuration

Known configuration fields include:

~~~text
BeaconPeriod
MaxStaNum
Enable2040Coex
Channel
TxPowerLevel
SSID
BroadcastSSID
Protocol
KeyIndex
Key_0
PSK
PwkCipherWPA
PwkCipherWPA2
GwkCipher
MCAST_REKEY
UCAST_REKEY
[WLAN_BEGIN]
[WLAN_END]
~~~

## 2.4 DHCP / DNS

dnsmasq provides the AP-side DHCP/DNS infrastructure.

The exact CarPlay-specific DHCP/DNS requirements have not been established and must not be invented.

## 2.5 mDNS

The mDNS consumer is now proven inside `mdnsd`: `SetupOneInterface()` reads `MDNS_DIRECTLINK_IFACE` with `getenv()` and participates in dedicated direct-link interface registration. The unresolved CarPlay question is now:

~~~text
Does the existing mDNS path advertise/discover CarPlay over uap0,
and what process selects the interface?
~~~

The high-value boundary is:

~~~mermaid
flowchart LR
    UAP["uap0"]
    MDNS["mDNS"]
    ENV["MDNS_DIRECTLINK_IFACE"]
    DIO["dio_manager"]
    AIRPLAY["libairplay.so"]

    UAP --> MDNS
    ENV --> DIO
    MDNS --> DIO
    DIO --> AIRPLAY
~~~

## 2.6 PF / routing

PF handles filtering/NAT/routing/queues around MHI2 network interfaces.

The remaining task is to correlate actual Wireless CarPlay traffic with the rules governing uap0.

## 2.7 WLAN / Bluetooth coexistence

AP startup invokes:

~~~text
uaputl coex_config /eso/telephone/coex.cfg
~~~

Known configuration includes:

~~~text
APBTCoex=0
acl_config enabled=1 btTime=40 wlanTime=60
~~~

This proves an existing coexistence mechanism. It does not prove a CarPlay-specific configuration.

---

# 3. Component / Subsystem Breakdown

## 3.1 io-sdiorm-mib2

Host-side SDIO resource manager for the shared Marvell controller.

## 3.2 devnp-mrvl_wlan-sdiorm.so

The production WLAN network-driver component.

Relevant driver symbols observed in the broader investigation include:

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
~~~

## 3.3 connectionmanager

Policy/orchestration layer above the Marvell AP control interface.

Responsibilities mapped include:

- AP activation;
- WLAN configuration;
- scan/connect/disconnect;
- password/PSK generation;
- channel/TX-power configuration;
- AP mode/power management;
- invoking uaputl.

## 3.4 uaputl

Direct Marvell AP firmware control utility.

Relevant commands include:

~~~text
sys_config
bss_config
bss_start
bss_stop
sys_cfg_ssid
sys_cfg_protocol
sys_cfg_auth
sys_cfg_wpa_passphrase
sys_cfg_pwk_cipher
sys_cfg_channel
sys_cfg_channel_ext
sys_cfg_scan_channels
sys_cfg_bcast_ssid_ctl
sys_cfg_max_sta_num
sys_cfg_tx_power
sys_cfg_radio
sys_cfg_ap_mac_address
sta_list
sta_deauth
coex_config
hostcmd
deepsleep
aggrpriotbl
~~~

## 3.5 uap0

Key wireless interface for the Wireless CarPlay target.

Known:

~~~text
10.173.189.1/24
DHCP range 10.173.189.10-99
~~~

## 3.6 dnsmasq

Provides DHCP/DNS for the AP environment.

## 3.7 mDNS / Bonjour

The AirPlay layer contains Bonjour APIs including DNSServiceRegister, DNSServiceUpdateRecord, DNSServiceGetAddrInfo and DNSServiceQueryRecord.

The WLAN investigation must meet the AirPlay investigation at the interface-selection boundary.

## 3.8 PF

PF is part of the MHI2 networking substrate. The important CarPlay questions are whether it permits:

- phone-to-MMI traffic on uap0;
- multicast/mDNS;
- CarPlay control traffic;
- CarPlay media streams;
- required peer-to-peer traffic.

## 3.9 WLAN / BT coexistence

~~~mermaid
flowchart LR
    BT["Bluetooth"]
    WLAN["WLAN"]
    COEX["Marvell coexistence"]

    BT <--> COEX
    WLAN <--> COEX
~~~

Wireless CarPlay requires both transports concurrently, making this existing mechanism important.

---

# 4. Evidence Status

## Proven

- Marvell 8787 provides shared WLAN/BT hardware.
- Production WLAN/BT firmware exists.
- io-sdiorm-mib2 is part of the SDIO architecture.
- devnp-mrvl_wlan-sdiorm.so is the WLAN driver.
- connectionmanager contains a networking_4WLAN implementation.
- uaputl controls the Marvell AP.
- uap0 exists as the AP-side interface.
- uap0 uses 10.173.189.1/24.
- DHCP range 10.173.189.10-99 exists.
- dnsmasq provides DHCP/DNS infrastructure.
- PF is part of the network architecture.
- mDNS infrastructure exists.
- WLAN/Bluetooth coexistence configuration exists.
- WLAN infrastructure is independent of carplay0.

## Partially traced

- exact SDIO startup sequence;
- exact firmware-load sequence;
- connectionmanager → uaputl runtime call sequence;
- AP state transitions;
- PF rule path for CarPlay traffic;
- production boot-time propagation of `MDNS_DIRECTLINK_IFACE` into `mdnsd`;
- final mDNS/AirPlay socket/interface binding;
- peer-to-peer routing/firewall behaviour.

## Not yet proven

- that stock CarPlay uses uap0;
- that uap0 is currently passed to AirPlay;
- that the recovered mDNS direct-link path can be redirected to uap0 without further adaptation;
- that MDNS_DIRECTLINK_IFACE=uap0 alone is sufficient;
- that all required Wireless CarPlay traffic is permitted;
- that the existing AP policy exactly matches Apple's Wireless CarPlay requirements.

---

# 5. End State / Trace Objective

~~~mermaid
flowchart TB
    PHONE["iPhone"]

    RADIO["Marvell 8787 WLAN"]
    DRIVER["WLAN driver"]
    UAP["uap0"]

    DHCP["dnsmasq"]
    MDNS["mDNS"]
    PF["PF"]

    DIO["dio_manager"]
    AIRPLAY["libairplay"]

    PHONE <--> RADIO
    RADIO --> DRIVER
    DRIVER --> UAP
    UAP --> DHCP
    UAP --> MDNS
    UAP --> PF
    MDNS --> DIO
    DIO --> AIRPLAY
~~~

The critical missing proof is:

~~~text
uap0
  ↓
mDNS / AirPlay
  ↓
DIO
  ↓
Wireless CarPlay session
~~~

---

# 6. Wireless CarPlay Integration Boundary

Current USB path:

~~~text
iPhone
  ↓
USB
  ├── /dev/ipod0
  └── carplay0
       ↓
      DIO
~~~

Target wireless path:

~~~text
iPhone
  ├── Bluetooth
  │     ↓
  │   iAP2 bootstrap
  │     ↓
  │   DIO
  │
  └── Wi-Fi
        ↓
       uap0
        ↓
       mDNS
        ↓
       AirPlay / DIO
~~~

The Wi-Fi subsystem therefore provides the IP transport/discovery substrate rather than becoming a separate CarPlay stack.

---

# 7. Highest-Value Next Traces

1. Find every consumer of MDNS_DIRECTLINK_IFACE.
2. Trace uap0 into mDNS socket/interface selection.
3. Trace AirPlay SetIFName, SetTransportType and multicast-interface setup from DIO.
4. Capture phone association and correlate DHCP, mDNS and DIO events.
5. Correlate PF rules with observed CarPlay packet flow.
6. Verify concurrent Bluetooth/WLAN operation during a CarPlay session.
7. Correlate Wi-Fi association with the Bluetooth/iAP2 bootstrap.

---

# 8. Evidence Discipline

### Proven
Directly supported by firmware, configuration, binary symbols or runtime evidence.

### Partially Traced
The components exist and the architectural relationship is established, but the complete runtime path is not mapped.

### Not Yet Proven
The proposed relationship is technically plausible but requires MHI2-specific evidence.

No external Wireless CarPlay implementation is treated as proof of MHI2 behaviour.


## 3.7 iAP2 Wi-Fi transport capability

The MU0678 ipod-drvr-iap2.so binary contains a dedicated Wi-Fi transport Identify handler, ident_info_tspwifi, and a static descriptor, sparams_id_info_wifitspcomp. Its fields include TransportComponentName, TransportSupportsiAP2Connection and TransportSupportsCarPlay.

The same binary contains the generic iAP2 transport callback layer: transport_send_pkt, transport_receive and transport_get_link_params.

This proves that the MU0678 iAP2 implementation contains an explicit Wi-Fi/iAP2/CarPlay transport capability model.

It does not prove that the existing uap0 network is already used for iAP2 traffic. The shipped /etc/mm/iap2.cfg selects Lightning Connector, so production wireless transport selection remains unresolved.

~~~text
uap0 exists
      +
iAP2 Wi-Fi capability exists
      !=
production Wireless CarPlay iAP2 is active
~~~

**Evidence:** E-032, E-034, E-036.
