# MHI2 AirPlay Architecture

Evidence-driven map of the AirPlay receiver layer used by MHI2 CarPlay, focused on interface selection, mDNS and the transport boundary required for Wireless CarPlay.

> **Status:** Active research  
> **Platform:** Audi MHI2 / QNX  
> **Target:** Wireless CarPlay

---

# 1. Current Mapped Architecture

## 1.1 AirPlay integration boundary

DIO contains direct AirPlay receiver/session references.

~~~mermaid
flowchart TB
    DIO["dio_manager"]
    SERVER["AirPlayReceiverServer"]
    SESSION["AirPlayReceiverSession"]
    SETUP["AirPlayReceiverSessionSetup"]
    SCREEN["AirPlayReceiverSessionScreen"]
    MODE["AirPlayReceiverSessionChangeModes"]
    SECURITY["AirPlayReceiverSessionSetSecurityInfo"]
    LIB["libairplay.so"]

    DIO --> SERVER
    DIO --> SESSION
    SESSION --> SETUP
    SESSION --> SCREEN
    SESSION --> MODE
    SESSION --> SECURITY
    DIO --> LIB
~~~

## 1.2 Bonjour / mDNS

AirPlay contains:

~~~text
DNSServiceRegister
DNSServiceUpdateRecord
DNSServiceGetAddrInfo
DNSServiceQueryRecord
_airplay._tcp.
~~~

~~~mermaid
flowchart LR
    AIRPLAY["libairplay.so"]
    BONJOUR["Bonjour / mDNS"]
    SERVICE["_airplay._tcp."]

    AIRPLAY --> BONJOUR
    BONJOUR --> SERVICE
~~~

This proves AirPlay participates in Bonjour service discovery/advertisement.

## 1.3 Interface selection

AirPlay contains:

~~~text
SocketSetBoundInterface
SocketSetPacketReceiveInterface
SocketSetMulticastInterface
IsWiFiNetworkInterface
IsUSBNetworkInterface

AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
~~~

~~~mermaid
flowchart TB
    DIO["dio_manager"]
    SCREEN["AirPlay Screen"]
    IFNAME["SetIFName"]
    TRANSPORT["SetTransportType"]
    MAC["SetClientIfMACAddr"]
    BOUND["SocketSetBoundInterface"]
    RX["SocketSetPacketReceiveInterface"]
    MCAST["SocketSetMulticastInterface"]

    DIO --> SCREEN
    SCREEN --> IFNAME
    SCREEN --> TRANSPORT
    SCREEN --> MAC
    SCREEN -.-> IFNAME
    SCREEN -.-> TRANSPORT
    SCREEN -.-> MAC
    RX --> MCAST
~~~

The library has explicit interface-selection machinery, but the apparent screen configuration exports are not the active selector in the production build. The recovered active Bonjour path uses the AirPlay object's `interfaceName` field.

## 1.4 Recovered Bonjour interface-selection path

In the production `libairplay.so`, `_UpdateBonjourAirPlay` reads an `interfaceName` field at `object + 0x6c`. If the field is non-empty, it calls `if_nametoindex(interfaceName)` and passes the resulting interface index to `DNSServiceRegister()`. If empty, the interface index is `0`.

Therefore the recovered path is:

```text
AirPlay object
  +0x6c interfaceName
        ↓
if_nametoindex()
        ↓
DNSServiceRegister(..., interfaceIndex, ...)
```

The production exports `AirPlayReceiverSessionScreen_SetIFName`, `SetTransportType` and `SetClientIfMACAddr` are no-op stubs, and `SocketSetBoundInterface` is a tiny stub/constant-return. They should not be treated as the active transport/interface adaptation point.

The substantive lower-level helpers remain `SocketSetPacketReceiveInterface`, `SocketSetMulticastInterface` and `IsWiFiNetworkInterface`; their callers and arguments still require tracing.

**Evidence:** E-018, E-019, E-026, E-027. See TRACE-004 and TRACE-005.

## 1.5 Current USB network path

Production:

~~~text
MDNS_DIRECTLINK_IFACE=carplay0
carplay0
  ↓
devnp-usbdnet.so
  ↓
USB
~~~

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    USB["USB"]
    NCM["devnp-usbdnet.so"]
    CARPLAY0["carplay0"]
    ENV["MDNS_DIRECTLINK_IFACE"]
    AIRPLAY["libairplay"]

    PHONE <--> USB
    USB --> NCM
    NCM --> CARPLAY0
    CARPLAY0 -.-> ENV
    ENV -.-> AIRPLAY
~~~

## 1.5 Wireless target

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    WIFI["Wi-Fi"]
    UAP["uap0"]
    MDNS["mDNS"]
    DIO["dio_manager"]
    AIRPLAY["libairplay"]

    PHONE <--> WIFI
    WIFI --> UAP
    UAP --> MDNS
    MDNS --> DIO
    DIO --> AIRPLAY
~~~

This is the target architecture, not a claim of completed Wireless CarPlay.

---

# 2. What Remains to Be Traced

## 2.1 MDNS_DIRECTLINK_IFACE

Highest-value configuration boundary:

~~~text
MDNS_DIRECTLINK_IFACE=carplay0
~~~

Find:

- every string reference;
- configuration parser;
- consumer process;
- propagation mechanism;
- socket/interface binding;
- AirPlay setup call.

~~~mermaid
flowchart LR
    CONFIG["MDNS_DIRECTLINK_IFACE"]
    CONSUMER["Consumer"]
    IFACE["Interface name"]
    SOCKET["Socket binding"]
    AIRPLAY["AirPlay"]

    CONFIG --> CONSUMER --> IFACE --> SOCKET --> AIRPLAY
~~~

## 2.2 DIO → AirPlay interface configuration

The screen setter exports are production no-ops and are not the active adaptation target.

Trace instead:

~~~text
where AirPlay interfaceName is populated
SocketSetPacketReceiveInterface
SocketSetMulticastInterface
IsWiFiNetworkInterface
~~~

Record:

~~~text
caller:
ifname:
transportType:
clientIfMAC:
session:
call timing:
~~~

## 2.3 Wi-Fi vs USB detection

Trace callers of:

~~~text
IsWiFiNetworkInterface
IsUSBNetworkInterface
~~~

Determine:

- interfaces tested;
- whether uap0 is accepted;
- whether carplay0 is preferred;
- selected transport value;
- whether transport selection affects setup/security.

## 2.4 Bonjour registration

The DNS-SD calls are now established as a real library/process boundary: `libairplay.so` calls `DNSServiceRegister`/related APIs through `libdns_sd.so`, with `mdnsd` providing the system mDNS implementation. See TRACE-003.

Trace:

~~~text
DNSServiceRegister
DNSServiceUpdateRecord
~~~

Determine:

- interface index;
- address;
- port;
- TXT records;
- registration timing;
- unregister behaviour.

## 2.5 AirPlay SETUP

Trace:

~~~text
AirPlayReceiverSessionSetup
~~~

and correlate:

~~~text
SETUP
session creation
screen creation
transport selection
security setup
network sockets
~~~

---

# 3. Component / Subsystem Breakdown

## 3.1 libairplay.so

Relevant APIs include:

~~~text
AirPlayReceiverServerCreateWithConfigFilePath
AirPlayReceiverServer
AirPlayReceiverSession
AirPlayReceiverSessionSetup
AirPlayReceiverSessionScreen
AirPlayReceiverSessionChangeModes
AirPlayReceiverSessionSetSecurityInfo
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
~~~

The library also contains Bonjour and network-interface helpers.

## 3.2 AirPlay server

~~~mermaid
flowchart TB
    CONFIG["AirPlay configuration"]
    SERVER["AirPlayReceiverServer"]
    SESSION["AirPlayReceiverSession"]

    CONFIG --> SERVER
    SERVER --> SESSION
~~~

The image references:

~~~text
/etc/airplay.conf
~~~

The reference does not prove the file is present on the dumped production filesystem.

## 3.3 AirPlay session

~~~mermaid
flowchart TB
    SESSION["AirPlayReceiverSession"]
    SETUP["SETUP"]
    SCREEN["Screen"]
    AUDIO["Audio"]
    NTP["NTP"]
    HID["Control / HID"]

    SESSION --> SETUP
    SESSION --> SCREEN
    SESSION --> AUDIO
    SESSION --> NTP
    SESSION --> HID
~~~

Exact MHI2 callback/media ownership remains to be mapped.

## 3.4 Screen interface configuration

The production exports:

~~~text
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
~~~

are no-op stubs and are therefore retired as candidates for the active USB → Wi-Fi adaptation boundary.

The recovered interface-selection path is instead:

~~~text
AirPlay object
  +0x6c interfaceName
        ↓
if_nametoindex()
        ↓
DNSServiceRegister(..., interfaceIndex, ...)
~~~

The active investigation targets are:

~~~text
where interfaceName is populated
SocketSetPacketReceiveInterface()
SocketSetMulticastInterface()
IsWiFiNetworkInterface()
~~~

The callers, arguments and final socket/interface binding remain unresolved. No patch should be made until those values and call paths are recovered.

## 3.5 Socket interface selection

Lower-level helpers:

~~~text
SocketSetBoundInterface
SocketSetPacketReceiveInterface
SocketSetMulticastInterface
~~~

Exact call graph remains to be traced.

## 3.6 Bonjour

AirPlay contains:

~~~text
DNSServiceRegister
DNSServiceUpdateRecord
DNSServiceGetAddrInfo
DNSServiceQueryRecord
_airplay._tcp.
~~~

This is the AirPlay discovery/advertisement boundary.

---

# 4. Evidence Status

## Proven

- MHI2 contains libairplay.so.
- DIO contains AirPlay receiver/session symbols.
- AirPlay contains Bonjour APIs.
- AirPlay references _airplay._tcp.
- AirPlay contains Wi-Fi/USB interface detection helpers.
- AirPlay contains explicit interface-binding functions.
- AirPlay Bonjour registration derives an interface index from its `interfaceName` field.
- The production screen interface/transport/client-MAC setters are no-op stubs.
- `SocketSetBoundInterface` is not the active production selector.
- AirPlay exposes screen IFName/transport/client-MAC APIs.
- Production CarPlay uses carplay0 for its direct-link network configuration.
- carplay0 is created by USB NCM infrastructure.

## Partially traced

- DIO → AirPlay session creation;
- screen transport configuration;
- mDNS interface selection;
- AirPlay socket binding;
- Bonjour TXT construction;
- session SETUP parameter flow.

## Not yet proven

- where the AirPlay `interfaceName` field is populated;
- callers/arguments of packet/multicast interface helpers;
- boot-time propagation of `MDNS_DIRECTLINK_IFACE` into `mdnsd`;
- whether the recovered interface path accepts `uap0` without modification;
- whether changing `MDNS_DIRECTLINK_IFACE` is sufficient;
- whether `libairplay.so` itself requires patching.

---

# 5. End State / Trace Objective

~~~mermaid
sequenceDiagram
    participant P as iPhone
    participant W as uap0
    participant M as mDNS
    participant D as dio_manager
    participant A as libairplay
    participant C as CarPlay session

    P->>W: Wi-Fi association
    P->>M: discovery
    M->>D: CarPlay discovery
    D->>A: create/configure receiver
    D->>A: interface + transport setup
    A->>C: session
    P->>A: SETUP / media / control
    A->>C: screen/audio/control
~~~

Completion requires exact evidence for every transition.

---

# 6. Wireless CarPlay Boundary

Current:

~~~text
MDNS_DIRECTLINK_IFACE=carplay0
        ↓
USB NCM
        ↓
CarPlay / AirPlay
~~~

Target candidate:

~~~text
MDNS_DIRECTLINK_IFACE=uap0
        ↓
Wi-Fi AP
        ↓
CarPlay / AirPlay
~~~

The word candidate is deliberate: the correct adaptation point may be DIO or AirPlay interface selection rather than the environment variable itself.

---

# 7. Highest-Value Next Traces

1. Prove the boot-time environment provenance of `MDNS_DIRECTLINK_IFACE` after the `mdnsd` consumer itself has been established.
2. Recover where AirPlay `interfaceName` is populated.
3. Trace callers and arguments of packet/multicast interface helpers.
4. Trace IsWiFiNetworkInterface and IsUSBNetworkInterface.
5. Trace socket interface binding.
6. Trace Bonjour registration and TXT construction.
7. Correlate AirPlay SETUP with interface selection.
8. Only then decide whether libairplay.so needs modification.

---

# 8. Evidence Discipline

Do not patch libairplay.so merely because Wireless CarPlay is the goal.

First prove:

~~~text
DIO
 ↓
current interface/transport arguments
 ↓
AirPlay interface selection
 ↓
socket binding
 ↓
mDNS / SETUP
~~~

If the existing AirPlay interface machinery accepts uap0, the smallest implementation may be outside the AirPlay library.
