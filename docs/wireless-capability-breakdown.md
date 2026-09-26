# MHI2 Wireless Capability Breakdown

This document records the reverse-engineered wireless/hotspot architecture of the MHI2 system. It is intentionally evidence-driven: conclusions are based on the firmware contents and configuration traced during the investigation. Unproven areas are explicitly called out rather than inferred.

## 1. The complete proven data path

The actual hotspot packet path is:

```
                     MMI / HMI
                         │
                         │ Connectivity / WLAN control
                         ▼
              connectivity_launcher
                         │
                         ▼
                connectionmanager
                         │
                         │ WLAN configuration
                         ▼
                    /tmp/uaputl.cfg
                         │
                         ▼
                      uaputl
                         │
                         ▼
                  Marvell 8787 WLAN FW
                         │
                         │
                    SDIO / io-sdiorm-mib2
                         │
                         ▼
                       uap0
                    10.173.189.1/24
                         │
                         ▼
                    Wi-Fi clients
                         │
                         ▼
                      dnsmasq
                   DHCP + DNS forwarder
                         │
                         ▼
                         PF
                    /etc/pf.conf
                   ┌──────┼──────┐
                   ▼      ▼      ▼
                  ppp0   ecm0   en0
                   │      │
                   ▼      ▼
                cellular USB/
                 NAD    tethering
                   │
                   ▼
                Internet
```

The key point is that **`uap0` is the actual hotspot/AP interface**.

`mlan0` is something completely different.

---

## 2. The Wi-Fi hardware stack

The hardware is a **Marvell 8787** WLAN device.

The dump contains:

```
advanced/MU0678-appimg/var/FwImage/
    sd8787_uapsta.bin
    w8787_wlan_SDIO_bt_SDIO.bin
```

and:

```
/armle/sbin/io-sdiorm-mib2
/armle/sbin/uaputl
/armle/sbin/mlan_region
/lib/dll/devnp-mrvl_wlan-sdiorm.so
```

Startup explicitly powers/resets the WLAN:

```
echo 1 > /dev/nvgpio/wlan_reset
```

and starts the SDIO resource manager:

```
/armle/sbin/io-sdiorm-mib2 \
    -p33 \
    -c45000000 \
    -h ioport=0x78000200,irq=47,inpclk=45000000 \
    -d nowlan_reset &
```

Then the WLAN firmware is loaded:

```
/usr/sbin/mvload \
    -p /mnt/app/var/FwImage \
    -a /eso/bin/PhoneCustomer
```

and the QNX network driver is mounted:

```
mount -Tio-pkt \
    -o drv_mode=3,max_uap_bss=1,eeprom_nmacs=3,force_shutdown,scan_chan_times=150:50:50 \
    /lib/dll/devnp-mrvl_wlan-sdiorm.so
```

So the hardware boundary is:

```
Tegra/QNX
   │
   ├── io-sdiorm-mib2
   ├── devnp-mrvl_wlan-sdiorm.so
   └── mvload
          │
          ▼
      Marvell 8787
          │
          ├── WLAN firmware
          └── Bluetooth firmware
```

This is not Linux/mac80211. It is a QNX `io-pkt` network-driver architecture.

---

## 3. The most important distinction: `uap0` vs `mlan0`

Startup makes this unambiguous.

### Hotspot/AP

```
if_up -p -r 100 uap0

ifconfig uap0 up
ifconfig uap0 mediaopt hostap
ifconfig uap0 10.173.189.1 netmask 255.255.255.0
```

Therefore:

**`uap0` = access-point interface.**

It gets:

```
10.173.189.1/24
```

and is explicitly put into:

```
mediaopt hostap
```

### Wi-Fi client

Separately:

```
if_up -p -r 10 mlan0

ifconfig mlan0 up

echo "ctrl_interface=/var/run/wpa_supplicant
ap_scan=2
update_config=1

" >/ramdisk/wpa_supplicant.conf

wpa_supplicant -Bi mlan0 \
    -c /ramdisk/wpa_supplicant.conf
```

Therefore:

**`mlan0` = station/client interface.**

And that is why the firmware contains `wpa_supplicant`.

It is **not evidence that wpa_supplicant implements the hotspot**.

The hotspot uses the Marvell AP/UAP interface and `uaputl`.

---

## 4. What actually creates the hotspot

The static startup code establishes the AP interface but does **not** contain the SSID/password.

Instead, the telephone/connectivity configuration contains:

```
"connectionmanager": {
    "autoProfilePath": "/eso/telephone/autoconf.xml",
    "customerProfilePath": "/eso/bin/customer_apn/autoconf.xml",
    "uaputlCfgPath": "/tmp/uaputl.cfg",
    "onlineIdleTimeInS": 300
}
```

The firmware therefore gives the chain:

```
connectionmanager
        │
        └── uaputlCfgPath
                │
                ▼
          /tmp/uaputl.cfg
```

So the AP configuration is **runtime-generated/state-derived**, rather than being a simple static `hostapd.conf`.

That explains why there is no conventional:

```
hostapd.conf
ssid=...
wpa_passphrase=...
```

architecture.

There is no hostapd here.

---

## 5. `start-ap.sh` proves the lower-level control mechanism

The telephone subsystem contains:

```
/eso/telephone/start-ap.sh
```

and it directly calls:

```
uaputl bss_stop
uaputl sys_config $2
```

then:

```
uaputl hostcmd ...
uaputl deepsleep 0
uaputl coex_config /eso/telephone/coex.cfg
uaputl aggrpriotbl ...
```

So the actual AP configuration operation is:

```
telephone / connectionmanager
            │
            ▼
         uaputl config
            │
            ▼
       Marvell WLAN FW
```

`uaputl` is consequently the firmware's **control interface to the Marvell WLAN firmware**.

---

## 6. DHCP is completely mapped

The AP network is:

```
10.173.189.0/24
```

with MMI:

```
10.173.189.1
```

DHCP comes from:

```
/eso/bin/apps/dnsmasq
```

The configuration is:

```
/etc/dnsmasq.conf
```

and explicitly says:

```
interface=lo0
interface=uap0
```

DHCP range:

```
10.173.189.10 - 10.173.189.99
```

with:

```
netmask 255.255.255.0
lease 3m
```

There is also:

```
dhcp-range=10.173.189.0,static,5m
```

and:

```
dhcp-script=/etc/dhcp/lease_changed.sh
```

Leases are persisted at:

```
/ramdisk/dnsmasq.leases
```

So:

```
Wi-Fi client
     │
     │ DHCPDISCOVER
     ▼
    uap0
     │
     ▼
   dnsmasq
     │
     ├── DHCP address
     ├── DHCP options
     └── lease tracking
```

---

## 7. DNS is also dnsmasq

The same daemon provides DNS.

It reads:

```
/etc/defaultdns.conf
/tmp/ppp0.resolv.conf
/tmp/ecm1.resolv.conf
/tmp/dhcp.resolv.conf
```

The architecture is:

```
Wi-Fi client
     │
     ├── DHCP ─────► dnsmasq
     │
     └── DNS :53 ──► dnsmasq
                         │
                         ▼
                    upstream DNS
```

The upstream DNS is dynamically supplied by the cellular connection.

There is a fallback:

```
8.8.8.8
208.67.222.222
```

in PF's `dns_server` table.

---

## 8. The Internet side is not Wi-Fi

The hotspot does not itself provide Internet.

It is a router/NAT endpoint.

The external side can be:

```
ppp0
```

or:

```
ecm0
```

PF explicitly defines:

```
ppp_if = "ppp0"
ecm_if = "ecm0"
wlan_if = "uap0"
```

and the NAT rules say:

```
nat pass on $ppp_if tagged WLAN -> ($ppp_if)
nat pass on $ecm_if tagged WLAN -> ($ecm_if)
```

So:

```
uap0
  │
  │ tagged WLAN
  ▼
 PF
  │
  ├── ppp0
  └── ecm0
```

This is hard evidence that the hotspot is an actual routed/NATed network.

---

## 9. Cellular modem architecture

The cellular side is another separate subsystem.

Startup launches:

```
/eso/bin/apps/TelitStarter
```

and the connectivity framework launches:

```
/eso/bin/apps/nad
```

The NAD configuration says:

```
"nad": {
    "path": "/dev/NAD/AT_port"
}
```

The logging configuration explicitly exposes:

```
CON_NAD_CINTERION
CON_NAD_MODULE_CINTERIONLTE
CON_NAD_MODULE_HUAWEI
CON_NAD_MODULE_NVBRUCE
CON_NAD_MODULE_SIERRA
```

So the architecture is deliberately abstracted around a **NAD (Network Access Device)**.

The firmware supports multiple modem implementations.

For the data connection, the connectivity configuration explicitly defines three possible mechanisms:

- ECM
- PPPd
- RMNET

Specifically:

```
"networkDataAccess": {
    "ecm": {
        "path": "",
        "timeout": 10
    },
    "pppd": {
        "connect": "/ramdisk/modem_scripts/modem_connect.sh",
        "disconnect": "/ramdisk/modem_scripts/modem_disconnect.sh",
        "pppdRunningCondition": "/var/run/ppp0.pid",
        "timeout": 30
    },
    "rmnet": {
        "path": "",
        "timeout": 10
    }
}
```

That is an important architectural boundary:

```
NAD abstraction
      │
      ├── ECM
      │     └── ecm0
      │
      ├── PPP
      │     └── ppp0
      │
      └── RMNET
```

The particular production path can therefore depend on the modem/platform configuration.

---

## 10. PPP path is fully traceable

When PPP comes up:

```
pppd
 │
 ▼
/etc/ppp/ip-up
```

`ip-up`:

1. deletes the existing default route
2. adds the PPP peer as the default route
3. creates the PPP local-host route
4. installs DNS
5. adds DNS servers to PF's `dns_server` table
6. creates `/tmp/ppp_connected`

So:

```
NAD
 │
 ▼
pppd
 │
 ▼
ppp0
 │
 ├── default route
 ├── DNS
 └── PF dns_server table
```

Then:

```
uap0
   │
   ▼
 NAT
   │
   ▼
ppp0
   │
   ▼
cellular network
```

---

## 11. ECM is another complete path

The firmware also has explicit ECM handling.

`ecm-down` tracks:

```
ecm0
ecm1
```

and manipulates:

```
pfctl
dnsmasq
/tmp/ecm1-setroute.sh
/tmp/ecm1-delroute.sh
/tmp/ecm1.resolv.conf
```

It also restarts/reconfigures dnsmasq when the external connection changes.

Therefore the hotspot is designed to survive external-network changes:

```
cellular state changes
        │
        ▼
      NAD/ECM
        │
        ▼
 connectivity manager
        │
        ├── routes
        ├── DNS state
        ├── PF state
        └── dnsmasq state
```

---

## 12. `modWLANserver.sh` reveals another important control edge

This script distinguishes:

```
# 0 = no internet connection available
# 1 = internet connection available
```

It manipulates:

```
/tmp/dhcp.opts
```

and then:

```
slay -fs SIGHUP dnsmasq
```

So the DHCP server's advertised network parameters are deliberately changed depending on whether the vehicle currently has Internet connectivity.

That gives:

```
NAD Internet state
        │
        ▼
 modWLANserver.sh
        │
        ▼
 /tmp/dhcp.opts
        │
        ▼
 dnsmasq SIGHUP
        │
        ▼
 DHCP configuration changes
```

This is a strong indication that the DHCP server is part of the **connectivity state machine**, rather than just being a permanently static DHCP daemon.

---

## 13. The hotspot is firewall-isolated

PF is not simply doing NAT.

It deliberately prevents hotspot clients from directly reaching the external interfaces:

```
block in quick on $wlan_if \
    from any to {($ppp_if), ($ecm_if), ($dbg_if)}
```

The external interfaces themselves have:

```
block drop in quick
```

and outgoing-source restrictions.

So the intended security model is:

```
Wi-Fi client
    │
    │ allowed outbound flow
    ▼
 PF
    │
    ▼
 NAT
    │
    ▼
 cellular
```

rather than unrestricted L2/L3 access between hotspot, cellular, debug, and MMI networks.

---

## 14. Client isolation exists

PF contains:

```
block in quick on $wlan_if from any to ($wlan_if)
```

followed by:

```
pass in quick on $wlan_if \
    from ($wlan_if:network) \
    to ($wlan_if:network)
```

So there is explicit WLAN-to-WLAN filtering.

This is part of the hotspot's security policy.

---

## 15. The hotspot isn't limited to ordinary Internet traffic

PF exposes a substantial application-specific WLAN service surface.

Examples include:

```
21001    WLAN tracing
20001    GEMIB communication
30001    SWDL fallback
23100    video
23101    video control
22222-23 audio
21600-21605 SSL/control
25010    EXLAP
28500    EXLAP UDP
49152    UPnP
49153-49162 UPnP
```

The queues are also explicitly classified:

```
video
audio
control
high
medium
default
```

with a nominal WLAN ALTQ bandwidth of:

```
150 Mb/s
```

So this WLAN infrastructure was designed as a broader **vehicle connectivity transport**, not merely “share cellular Internet.”

---

## 16. UPnP is integrated

The connectivity launcher starts:

```
dev-upnp
```

and PF explicitly permits:

```
239.255.255.0/24
```

and:

```
49152
49153-49162
```

over WLAN.

The firmware also contains:

```
upnp_file_extensions.xml
libasimmxconnectivity_upnpproxy.so
```

So UPnP is another first-class connectivity service.

---

## 17. HMI architecture

The HMI logging configuration contains distinct WLAN layers:

```
hmi_App_Wlan_Main
hmi_App_Wlan_DSI
hmi_App_Wlan_HMI
```

alongside:

```
hmi_App_Connectivity_Main
hmi_App_Data_Main
hmi_App_Umts_Main
```

This gives the following top-level architecture:

```
                 HMI
                  │
         ┌────────┴─────────┐
         ▼                  ▼
  Connectivity HMI      Data/UMTS HMI
         │
         ▼
    WLAN HMI / DSI
         │
         ▼
 connectivity framework
```

There are also corresponding proxy libraries:

```
libasimmxwlanproxy.so
libasimmxDiagWlanTypesproxy.so
```

which establishes that the HMI communicates with the connectivity subsystem through the generated ASI/DSI interface layer.

---

## 18. Connectivity launcher is the supervisor

The production configuration identifies:

```
connectivity_launcher
```

as the process supervisor.

It manages:

```
telephone
bluetooth
connectionmanager
messaging
dev-upnp
btstack
nad
```

and gives `connectionmanager`:

```
preCondition = /tmp/uap0
```

This is a major architectural fact.

It means the startup dependency is:

```
WLAN initialization
       │
       ▼
/tmp/uap0 exists
       │
       ▼
connectionmanager allowed to start
```

Therefore `connectionmanager` is **above the raw WLAN driver but below the HMI/service layer**.

---

## 19. Telephone is also above the modem

The connectivity supervisor launches:

```
telephone
nad
```

separately.

The telephone configuration has extensive ASI/DSI proxies to:

```
NADServices
CallHandlingServices
HandsfreeServices
DSINAD
DSIMobileEquipment
```

So the modem is not directly controlled by the HMI.

The architecture is approximately:

```
HMI
 │
 ▼
DSI / ASI proxies
 │
 ├── telephone
 │
 ├── connectionmanager
 │
 └── NAD
      │
      ▼
 /dev/NAD/AT_port
      │
      ▼
    modem
```

---

## 20. Persistence is a separate layer

The hotspot enable/disable engineering scripts are revealing.

Enable:

```
/eso/bin/apps/pc b:0:3221356628:0.6 1
```

Disable:

```
/eso/bin/apps/pc b:0:3221356628:0.6 0
```

Wi-Fi Client HMI is separately:

```
/eso/bin/apps/pc b:0:3221356628:8.2 1
```

or:

```
/eso/bin/apps/pc b:0:3221356628:8.2 0
```

So there are **separate persistent configuration bits** for:

- WLAN Hotspot
- WLAN Client HMI

This is not the same thing as turning the radio on.

The SCALE scripts additionally show:

```
WLAN_Module
AK_WLAN_ONOFF
AK_CAR_config
```

which are RCC-side coding/configuration controls.

Therefore there are at least two configuration layers:

```
vehicle/RCC coding
       │
       ▼
persistent MMI configuration
       │
       ▼
runtime connectivity state
```

---

## 21. What `wpa_supplicant` is actually doing

The firmware starts:

```
wpa_supplicant -Bi mlan0
```

with a generated configuration:

```
/ramdisk/wpa_supplicant.conf
```

containing:

```
ctrl_interface=/var/run/wpa_supplicant
ap_scan=2
update_config=1
```

But `mlan0` is never assigned:

```
10.173.189.1
```

and never gets:

```
mediaopt hostap
```

Those operations are performed on `uap0`.

Therefore:

**`wpa_supplicant` is the Wi-Fi client-side component, not the hotspot AP security engine.**

The hotspot's AP authentication/configuration is below the connection manager, through `uaputl` → Marvell firmware.

That is one of the strongest architectural conclusions from the dump.

---

## 22. What is actually persistent vs runtime

The layers can now be separated.

### Persistent configuration

```
pc
 │
 ├── WLAN Hotspot enable
 └── WLAN Client HMI enable
```

plus RCC coding:

```
WLAN_Module
AK_WLAN_ONOFF
AK_CAR_config
```

### Runtime WLAN

```
io-sdiorm-mib2
       │
       ▼
Marvell driver
       │
       ▼
uap0 / mlan0
```

### Runtime AP configuration

```
connectionmanager
       │
       ▼
/tmp/uaputl.cfg
       │
       ▼
uaputl
       │
       ▼
Marvell firmware
```

### Runtime network services

```
dnsmasq
PF
routes
DNS state
```

### Runtime WAN

```
NAD
 │
 ├── ECM
 ├── PPP
 └── RMNET
```

---

## 23. What happens when Internet connectivity changes

The evidence gives us this state machine:

```
             NAD
              │
              ▼
        WAN connection
              │
        ┌─────┴─────┐
        │           │
       ECM         PPP
        │           │
       ecm0        ppp0
        │           │
        └─────┬─────┘
              ▼
       connectivity
          manager
              │
        ┌─────┴─────┐
        ▼           ▼
     routing       DNS
        │           │
        ▼           ▼
       PF         dnsmasq
        │           │
        └─────┬─────┘
              ▼
            uap0
              │
              ▼
         Wi-Fi clients
```

The scripts explicitly reconfigure routes, DNS, PF tables and dnsmasq when ECM/PPP state changes.

---

## 24. What we have NOT proven yet

There are a few pieces where no answer should be invented.

### A. Exact SSID/password generation

We have proven:

```
connectionmanager
      ↓
/tmp/uaputl.cfg
      ↓
uaputl
      ↓
Marvell firmware
```

But the static dump does **not** expose the contents of `/tmp/uaputl.cfg`.

It is a runtime file.

We therefore cannot yet truthfully say:

> “function X generates the SSID and function Y generates the WPA key”

without disassembling `connectionmanager`/`telephone` and tracing the file write.

### B. Exact `uaputl` command sequence for production hotspot startup

`start-ap.sh` proves several commands, but it takes its actual AP configuration from:

```
$2
```

and the production framework points to:

```
/tmp/uaputl.cfg
```

The missing link is the exact producer/consumer sequence for that file.

### C. Exact modem selection on this particular MU0678 production configuration

The framework supports:

```
ECM
PPP
RMNET
```

and multiple modem families.

We know the live dump starts:

```
TelitStarter
nad
```

and the logs show a Cinterion/Telit-related USB device, but the final production data-bearer selection should be established from the live configuration/state rather than assumed from the available modules.

---

## 25. The architectural map as it stands

The cleanest representation is:

```
                         ┌─────────────────────────┐
                         │          MMI/HMI         │
                         │ Connectivity/WLAN HMI   │
                         └────────────┬────────────┘
                                      │
                               ASI / DSI proxies
                                      │
                                      ▼
                       ┌─────────────────────────────┐
                       │   connectivity_launcher     │
                       │       service supervisor     │
                       └──────────────┬──────────────┘
                                      │
                 ┌────────────────────┼──────────────────────┐
                 │                    │                      │
                 ▼                    ▼                      ▼
          connectionmanager        telephone                 NAD
                 │                    │                      │
                 │                    │                      │
                 ▼                    └──────────────┐       │
          /tmp/uaputl.cfg                           │       │
                 │                                   ▼       ▼
                 ▼                                WLAN/BT /dev/NAD/AT_port
              uaputl                                    │
                 │                                       ▼
                 ▼                                     modem
          Marvell 8787 firmware
                 │
                 ▼
       devnp-mrvl_wlan-sdiorm
                 │
            ┌────┴────┐
            │         │
           uap0     mlan0
            │         │
            │         └──── wpa_supplicant
            │
            │ 10.173.189.1/24
            ▼
        Wi-Fi clients
            │
            ├──────── DHCP ─────────┐
            │                       ▼
            │                    dnsmasq
            │                       │
            │                       └── DNS
            │
            ▼
           PF
            │
            ├──── NAT ────► ppp0 ───► NAD/cellular
            │
            ├──── NAT ────► ecm0 ───► NAD/USB
            │
            └──── policy / isolation / service ports
```

This is the current evidence-based wireless/hotspot architecture.