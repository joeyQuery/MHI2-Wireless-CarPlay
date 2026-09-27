# MHI2 Wireless CarPlay — Technical English

[**Plain English**](../README.md) · **Technical English**

Reverse engineering MHI2 and CarPlay to understand and develop **Wireless Apple CarPlay** on **supported Audi MHI2 units**.

> **Status:** Research / reverse engineering  
> **Platform:** Audi MHI2 on MU0678-class firmware [QNX]

---

# Project Goal

MHI2 already contains substantial infrastructure relevant to Wireless CarPlay:

- Bluetooth
- iAP / iAP2
- Wi-Fi
- Marvell WLAN/BT hardware
- mDNS / Bonjour
- AirPlay
- CarPlay
- DIO / smartphone integration
- USB NCM networking

The project goal is to determine how these existing components can be connected to provide Wireless CarPlay **without replacing the MHI2 CarPlay stack with an unrelated implementation**.

The work is evidence-driven. Firmware contents, configuration, binaries, runtime behaviour, controlled experiments and protocol traces are treated as evidence. Unproven relationships are explicitly marked as such.

---

# Current Architecture

The current production CarPlay path is USB-derived.

~~~mermaid
flowchart TB
    PHONE["iPhone"]
    USB["USB"]
    IPOD["/dev/ipod0"]
    IAP2["iAP2"]
    NCM["devnp-usbdnet.so"]
    CARPLAY0["carplay0"]
    DIO["dio_manager"]
    AIRPLAY["libairplay.so"]
    CP["CarPlay session"]
    MMI["MHI2 MMI"]

    PHONE --> USB
    USB --> IPOD
    IPOD --> IAP2
    IAP2 --> DIO
    USB --> NCM
    NCM --> CARPLAY0
    CARPLAY0 --> DIO
    DIO --> AIRPLAY
    AIRPLAY --> CP
    CP --> MMI
~~~

The important finding is that the existing CarPlay implementation is not simply AirPlay over USB. There are separate iAP2 and network resources around the CarPlay session.

---

# Wireless CarPlay Target

The target architecture is to preserve the existing DIO / AirPlay / CarPlay stack while replacing the USB-specific transport dependencies. All edges in the target diagram are unresolved targets, not proven execution edges.

~~~mermaid
flowchart TB
    PHONE["iPhone"]
    BT["Bluetooth"]
    WIAP["Wireless iAP2"]
    WIFI["Wi-Fi"]
    UAP["uap0"]
    MDNS["mDNS / Bonjour"]
    DIO["dio_manager"]
    AIRPLAY["libairplay.so"]
    CP["CarPlay session"]
    MMI["MHI2 MMI"]

    PHONE -.-> BT
    BT -.-> WIAP
    WIAP -.-> DIO
    PHONE -.-> WIFI
    WIFI -.-> UAP
    UAP -.-> MDNS
    MDNS -.-> DIO
    DIO -.-> AIRPLAY
    AIRPLAY -.-> CP
    CP -.-> MMI
~~~

This is the **target architecture**, not a claim that every edge has already been proven on production MHI2.

---

# What We Know

## Wi-Fi

The MHI2 wireless subsystem contains an existing Marvell 8787 WLAN/BT platform.

Known components include:

~~~text
io-sdiorm-mib2
devnp-mrvl_wlan-sdiorm.so
connectionmanager
uaputl
dnsmasq
PF
~~~

The existing AP interface is:

~~~text
uap0
10.173.189.1/24
~~~

Known DHCP range:

~~~text
10.173.189.10 - 10.173.189.99
~~~

connectionmanager contains a dedicated networking_4WLAN implementation and invokes the Marvell uaputl/AP configuration path.

The WLAN and Bluetooth functions share the Marvell controller and have an existing coexistence configuration.

See **[Wi-Fi](docs/wifi.md)**.

## Bluetooth / iAP2

The production image contains substantial iAP/iAP2 infrastructure, including:

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

Production CarPlay currently uses the USB-derived /dev/ipod0 boundary.

The Bluetooth configuration contains:

~~~text
enableIap=false
~~~

The production configuration contains `enableIap=false` while substantial Bluetooth iAP/iAP2 implementation exists in the image. The exact functionality controlled by `enableIap` remains unresolved. Active-device callbacks reach `CIapBTChannel::updateiAPDevice()`, and the runtime-supplied endpoint reaches `open64()`; the endpoint value itself remains unresolved.

See **[Bluetooth](docs/bluetooth.md)** and **[iAP2](docs/iap2.md)**.

## AirPlay / mDNS

libairplay.so contains explicit networking and interface-selection infrastructure, including:

~~~text
DNSServiceRegister
DNSServiceUpdateRecord
DNSServiceGetAddrInfo
DNSServiceQueryRecord

SocketSetBoundInterface
SocketSetPacketReceiveInterface
SocketSetMulticastInterface

IsWiFiNetworkInterface
IsUSBNetworkInterface

AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
~~~

It also references:

~~~text
_airplay._tcp.
~~~

The production library's `_UpdateBonjourAirPlay` path now provides a concrete interface-selection edge: its `interfaceName` field is converted with `if_nametoindex()` and the resulting interface index is supplied to `DNSServiceRegister()`. The previously suspected screen setter APIs are production no-ops, so they are no longer treated as the presumed transport selector.

See **[AirPlay](docs/airplay.md)**.

## DIO / CarPlay

dio_manager is the central MHI2 CarPlay integration boundary identified so far.

Relevant functionality includes:

~~~text
requestCarPlay
checkCarPlayCompatibility

notifyiAP2DeviceConnected
notifyiAP2DeviceDisconnected
iAP2Connect

onEvent_sessionCreated
onEvent_sessionFinalized
onEvent_sessionCurrentModeChanged
onEvent_sessionChangeModeCompletion
~~~

DIO also contains direct AirPlay receiver/session integration.

This makes DIO the most important convergence point between:

~~~text
Bluetooth / iAP2
Wi-Fi / mDNS
AirPlay
CarPlay session state
~~~

See **[CarPlay](docs/carplay.md)**.

---

# The Two Current Networking Worlds

One of the important architectural findings is that MHI2 currently has two distinct network interfaces around CarPlay.

~~~mermaid
flowchart LR
    USB["USB"]
    NCM["devnp-usbdnet.so"]
    CP0["carplay0"]
    WLAN["Marvell WLAN"]
    UAP["uap0"]

    USB --> NCM --> CP0
    WLAN --> UAP
~~~

## carplay0

The existing USB CarPlay direct-link network.

Production configuration contains:

~~~text
MDNS_DIRECTLINK_IFACE=carplay0
~~~

## uap0

The existing MHI2 WLAN/AP interface.

~~~text
10.173.189.1/24
~~~

The objective is **not** to assume that uap0 can simply replace carplay0.

The actual interface-selection path must first be traced through DIO, mDNS and AirPlay.

---

# Highest-Value Unknowns

### 1. Bluetooth → iAP2

What exactly does:

~~~text
enableIap=false
~~~

disable?

Does enabling the existing Bluetooth iAP infrastructure produce the transport that DIO expects, or is additional adaptation required?

### 2. DIO's iAP2 transport boundary

Is the production /dev/ipod0 dependency:

- a configuration choice;
- a transport abstraction with a USB implementation;
- or a hard-coded USB dependency?

This must be resolved from the binary call path.

### 3. MDNS_DIRECTLINK_IFACE

`mdnsd` is now proven to consume:

~~~text
MDNS_DIRECTLINK_IFACE
~~~

through `getenv()` in `SetupOneInterface()`, with dedicated direct-link interface registration. What remains unproven is how the production boot path exports `MDNS_DIRECTLINK_IFACE=carplay0` into the running daemon and how that registered interface reaches final socket binding.

### 4. DIO → AirPlay interface selection

The three exported screen interface/transport/client-MAC setters are production no-op stubs. The active path recovered so far is the AirPlay object's `interfaceName` → `if_nametoindex()` → `DNSServiceRegister()` path. The next trace is to recover where `interfaceName` is populated and how packet/multicast socket helpers consume the selected interface.

### 5. Wi-Fi / AirPlay convergence

Can the existing AirPlay networking code operate on uap0, or does MHI2's DIO integration explicitly constrain it to carplay0?

### 6. Bluetooth + Wi-Fi correlation

How does the Bluetooth/iAP2 bootstrap become associated with the Wi-Fi/AirPlay session belonging to the same iPhone?

---

# Evidence Status

## Proven

- MHI2 contains a substantial WLAN/BT infrastructure.
- Marvell 8787 is used for WLAN/BT.
- uap0 exists as the WLAN/AP interface.
- connectionmanager contains WLAN/AP orchestration.
- uaputl controls the Marvell AP path.
- DHCP/DNS infrastructure exists through dnsmasq.
- PF is part of the networking architecture.
- WLAN/Bluetooth coexistence configuration exists.
- iAP/iAP2 infrastructure exists.
- Production CarPlay uses /dev/ipod0.
- Production CarPlay networking uses carplay0.
- MDNS_DIRECTLINK_IFACE=carplay0 exists in the production configuration.
- mdnsd explicitly consumes MDNS_DIRECTLINK_IFACE via getenv() and direct-link interface handling, but boot-time environment propagation remains unproven.
- libairplay.so contains Bonjour/mDNS functionality.
- libairplay.so contains Wi-Fi/USB interface-selection helpers.
- libairplay.so exposes screen interface/transport configuration APIs.
- DIO contains iAP2 and AirPlay/CarPlay integration functionality.
- Bluetooth iAP active-device callbacks supply a runtime endpoint to `CIapBTChannel`, which reaches `open64()`.
- AirPlay Bonjour registration converts its `interfaceName` to an interface index and passes it to `DNSServiceRegister()`.
- `mdnsd` explicitly consumes `MDNS_DIRECTLINK_IFACE` via `getenv()` and has direct-link interface handling.

## Partially traced

- Bluetooth → iAP2 runtime path;
- enableIap control flow;
- iAP2 → DIO callback chain;
- DIO's dependency on /dev/ipod0;
- MDNS_DIRECTLINK_IFACE consumer;
- DIO → AirPlay interface arguments;
- AirPlay socket/interface binding;
- mDNS → DIO discovery path;
- Bluetooth/iAP2 ↔ Wi-Fi/AirPlay session correlation.

## Not yet proven

- enableIap=true is sufficient for Wireless CarPlay;
- uap0 can simply replace carplay0;
- MDNS_DIRECTLINK_IFACE=uap0 is sufficient;
- DIO is completely transport-neutral;
- AirPlay requires no modification;
- the existing Bluetooth iAP infrastructure is already a working Wireless CarPlay bootstrap.

---

# Documentation

| Document | Scope |
|---|---|
| **[Wireless CarPlay Architecture](docs/wireless-carplay-architecture.md)** | Master end-to-end architecture, integration boundaries and evidence status |
| **[Bluetooth](docs/bluetooth.md)** | Bluetooth architecture, services and behaviour |
| **[iAP2](docs/iap2.md)** | iAP/iAP2 components and transport boundary |
| **[Wi-Fi](docs/wifi.md)** | Marvell WLAN, AP, uap0, connectionmanager, uaputl, DHCP/DNS, PF and coexistence |
| **[AirPlay](docs/airplay.md)** | AirPlay receiver, mDNS/Bonjour and interface selection |
| **[CarPlay](docs/carplay.md)** | DIO and the MHI2 CarPlay integration boundary |
| **[Repository Status](STATUS.md)** | Current research state, blockers and evidence rules |
| **[Roadmap](docs/roadmap.md)** | Dependency-oriented research and implementation roadmap |
| **[Evidence Register](docs/evidence.md)** | Cross-project evidence ledger |
| **[Disproven](docs/disproven.md)** | Eliminated interpretations and dead ends |
| **[Binary Inventory](docs/binaries.md)** | Binary/process/library inventory and trace status |
| **[Firmware Provenance](docs/firmware.md)** | Firmware baseline and provenance rules |
| **[Glossary](docs/glossary.md)** | Canonical project terminology |
| **[SSH Environment](docs/ssh-environment.md)** | Runtime investigation environment |
| **[Call Graphs](docs/call-graphs/README.md)** | Function-level execution-trace structure |
| **[Execution Traces](traces/README.md)** | Transport-boundary execution/data-flow traces |
| **[Source of Truth](docs/source-of-truth.md)** | Documentation authority hierarchy |
| **[Documentation Index](docs/README.md)** | Complete documentation map |

---

# Roadmap

The detailed dependency-oriented roadmap is maintained in **[docs/roadmap.md](docs/roadmap.md)**.

Current phase:

- establish the exact Bluetooth → iAP2 path;
- resolve the DIO `/dev/ipod0` transport boundary;
- resolve `MDNS_DIRECTLINK_IFACE` consumption;
- recover DIO → AirPlay interface/transport arguments;
- correlate Bluetooth/iAP2 and Wi-Fi/AirPlay into one session.

The repository deliberately does not mark Wireless CarPlay as implemented until those transport boundaries are supported by MHI2-specific evidence.

---

# Primary Reference

**[LIVI](https://github.com/f-io/LIVI)** is used as an open-source reference for Wireless CarPlay protocol research.

LIVI is a protocol/reference implementation, **not evidence of MHI2's internal implementation**.

---

# Safety

This project involves reverse engineering and modifying automotive infotainment firmware.

Always retain original files, hashes and a reliable recovery path before experimenting with a vehicle.

---

The documentation authority hierarchy is defined in [`docs/source-of-truth.md`](docs/source-of-truth.md). Execution traces in [`traces/`](traces/README.md) are the authoritative place for recovered call/data-flow edges; architectural diagrams may contain dashed target/inference edges and must not be read as proof.

> **Trace it. Prove it. Document it.**
