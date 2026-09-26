# MHI2 CarPlay Architecture

Evidence-driven map of the MHI2 CarPlay application boundary, connecting iAP2, mDNS, AirPlay and the existing USB transport to the Wireless CarPlay target.

> **Status:** Active research  
> **Platform:** Audi MHI2 / QNX  
> **Target:** Wireless CarPlay

---

# 1. Current Mapped Architecture

## 1.1 CarPlay application boundary

The key component is:

~~~text
dio_manager
~~~

It contains both iAP2 and AirPlay integration.

~~~mermaid
flowchart TB
    PHONE["iPhone"]
    IAP2["iAP2"]
    MDNS["mDNS / Bonjour"]
    DIO["dio_manager"]
    AIRPLAY["libairplay.so"]
    SESSION["CarPlay session"]

    PHONE --> IAP2
    PHONE --> MDNS
    IAP2 --> DIO
    MDNS --> DIO
    DIO --> AIRPLAY
    AIRPLAY --> SESSION
~~~

DIO is therefore the central integration boundary rather than AirPlay alone.

## 1.2 Existing USB CarPlay path

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    USB["USB"]
    IPOD["/dev/ipod0"]
    IAP2["iAP2"]
    NCM["devnp-usbdnet.so"]
    CARPLAY0["carplay0"]
    DIO["dio_manager"]
    AIRPLAY["libairplay.so"]

    PHONE --> USB
    USB --> IPOD
    IPOD --> IAP2
    IAP2 --> DIO
    USB --> NCM
    NCM --> CARPLAY0
    CARPLAY0 --> DIO
    DIO --> AIRPLAY
~~~

The production architecture has two USB-derived CarPlay resources:

~~~text
/dev/ipod0
carplay0
~~~

## 1.3 CarPlay state machine

Relevant DIO symbols:

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

Conceptual state flow:

~~~mermaid
flowchart LR
    DEVICE["iAP2 device"]
    CONNECT["iAP2Connect"]
    COMPAT["checkCarPlayCompatibility"]
    REQUEST["requestCarPlay"]
    CREATED["sessionCreated"]
    MODE["sessionCurrentModeChanged"]
    FINAL["sessionFinalized"]

    DEVICE --> CONNECT
    CONNECT --> COMPAT
    COMPAT --> REQUEST
    REQUEST --> CREATED
    CREATED --> MODE
    MODE --> FINAL
~~~

This is a state-machine map, not a claim that the exact call order has been fully recovered.

## 1.4 AirPlay boundary

DIO references:

~~~text
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

~~~mermaid
flowchart TB
    DIO["dio_manager"]
    SERVER["AirPlayReceiverServer"]
    SESSION["AirPlayReceiverSession"]
    SCREEN["AirPlayReceiverSessionScreen"]
    SETUP["AirPlayReceiverSessionSetup"]
    SECURITY["AirPlayReceiverSessionSetSecurityInfo"]

    DIO --> SERVER
    DIO --> SESSION
    SESSION --> SCREEN
    SESSION --> SETUP
    SESSION --> SECURITY
~~~

## 1.5 Current network identity

Production:

~~~text
carplay0
MDNS_DIRECTLINK_IFACE=carplay0
~~~

Wireless infrastructure:

~~~text
uap0
10.173.189.1/24
~~~

These are currently independent network worlds.

---

# 2. What Remains to Be Traced

## 2.1 iAP2 → DIO

Trace:

~~~mermaid
flowchart LR
    IAP2["iAP2"]
    DEVICE["iAP2 device event"]
    DIO["dio_manager"]
    COMPAT["CarPlay compatibility"]
    SESSION["CarPlay session"]

    IAP2 --> DEVICE --> DIO --> COMPAT --> SESSION
~~~

Recover exact event/callback, device object, state transition, compatibility result and session creation.

## 2.2 DIO → AirPlay

Trace all calls to:

~~~text
AirPlayReceiverServer
AirPlayReceiverSession
AirPlayReceiverSessionSetup
AirPlayReceiverSessionScreen
AirPlayReceiverSessionChangeModes
AirPlayReceiverSessionSetSecurityInfo
~~~

Record caller, arguments, return values, call order and error handling.

## 2.3 Screen transport configuration

Highest-value APIs:

~~~text
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
~~~

Recover the current values before changing them.

## 2.4 Network interface selection

Current:

~~~text
MDNS_DIRECTLINK_IFACE=carplay0
~~~

Target candidate:

~~~text
uap0
~~~

Potential boundaries:

~~~mermaid
flowchart LR
    ENV["MDNS_DIRECTLINK_IFACE"]
    DIO["DIO"]
    AIRPLAY["AirPlay"]
    SOCKET["Socket binding"]
    WIFI["uap0"]

    ENV -.-> DIO
    DIO -.-> AIRPLAY
    AIRPLAY -.-> SOCKET
    SOCKET -.-> WIFI
~~~

Dashed edges are unresolved.

## 2.5 Wireless bootstrap

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    BT["Bluetooth"]
    IAP["Wireless iAP2"]
    WIFI["Wi-Fi / uap0"]
    MDNS["mDNS"]
    DIO["DIO"]

    PHONE --> BT
    BT --> IAP
    IAP --> DIO
    PHONE --> WIFI
    WIFI --> MDNS
    MDNS --> DIO
~~~

The key task is correlating the two transports into the same CarPlay session.

---

# 3. Component / Subsystem Breakdown

## 3.1 DIO Manager

DIO integrates:

~~~text
smartphone integration
iAP2
mDNS
AirPlay
CarPlay session state
~~~

Relevant state-machine surface:

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

## 3.2 iAP2

Firmware components include:

~~~text
libiap2client.so
ipod-drvr-iap2.so
mss-ipodiap2.so
devu-iap2-tegra3-ci.so
devu-iap2ncm-tegra3-ci.so
iap2cli
~~~

Production DIO uses the USB-derived /dev/ipod0 boundary.

## 3.3 USB NCM

Network side:

~~~text
devnp-usbdnet.so
carplay0
~~~

This is the production direct-link network interface.

## 3.4 mDNS

Production configuration:

~~~text
MDNS_DIRECTLINK_IFACE=carplay0
~~~

AirPlay contains Bonjour APIs and:

~~~text
_airplay._tcp.
~~~

## 3.5 AirPlay

The receiver provides:

~~~text
server
session
SETUP
screen
audio
control
security
~~~

Exact MHI2 media callback ownership remains to be traced.

## 3.6 Media / HMI

The existing CarPlay session ultimately reaches the MHI2 media/HMI stack.

The Wireless CarPlay project should preserve this output path and change transport integration only.

---

# 4. Evidence Status

## Proven

- DIO is a CarPlay integration boundary.
- DIO contains iAP2 connection/state symbols.
- DIO contains AirPlay receiver/session symbols.
- Production CarPlay uses /dev/ipod0.
- Production CarPlay networking uses carplay0.
- carplay0 is provided by USB NCM infrastructure.
- MDNS_DIRECTLINK_IFACE=carplay0.
- AirPlay contains Wi-Fi/USB interface-selection helpers.
- MHI2 already contains a WLAN/AP subsystem at uap0.

## Partially traced

- exact iAP2 → DIO callback chain;
- exact DIO → AirPlay call sequence;
- exact screen transport values;
- exact mDNS consumer;
- exact network-interface binding;
- exact CarPlay media callback path.

## Not yet proven

- Wireless CarPlay is already implemented in DIO.
- DIO is transport-neutral at every iAP2 boundary.
- uap0 can simply replace carplay0.
- MDNS_DIRECTLINK_IFACE=uap0 is sufficient.
- AirPlay requires no changes.
- Existing Bluetooth iAP infrastructure is already usable for Wireless CarPlay.

---

# 5. End State / Trace Objective

~~~mermaid
sequenceDiagram
    participant P as iPhone
    participant B as Bluetooth
    participant I as iAP2
    participant W as uap0
    participant M as mDNS
    participant D as dio_manager
    participant A as libairplay
    participant C as CarPlay session

    P->>B: Bootstrap
    B->>I: iAP2 transport
    I->>D: Device/session event
    P->>W: Wi-Fi association
    W->>M: Network availability
    P->>M: CarPlay discovery
    M->>D: Discovery/event
    D->>D: Compatibility / requestCarPlay
    D->>A: Create/configure AirPlay session
    A->>C: Session
    P->>A: SETUP / media / control
    A->>C: CarPlay streams
~~~

Completion requires a timestamp-correlated trace of all major transitions.

---

# 6. Wireless CarPlay Implementation Boundary

Current:

~~~text
iPhone
 ├─ USB iAP2 → /dev/ipod0 → DIO
 └─ USB NCM → carplay0 → DIO/AirPlay
~~~

Target:

~~~text
iPhone
 ├─ Bluetooth → wireless iAP2 → DIO
 └─ Wi-Fi → uap0 → mDNS/AirPlay → DIO
~~~

The goal is a transport adaptation around the existing DIO/CarPlay stack, not a new CarPlay receiver.

---

# 7. Highest-Value Next Traces

1. Trace notifyiAP2DeviceConnected callers.
2. Trace iAP2Connect construction and transport.
3. Trace requestCarPlay and compatibility checks.
4. Trace AirPlay screen interface setters.
5. Resolve MDNS_DIRECTLINK_IFACE.
6. Correlate mDNS discovery with DIO events.
7. Correlate AirPlay SETUP with selected interface.
8. Verify screen/audio/control output after transport adaptation.

---

# 8. Evidence Discipline

This document is the integration map.

Detailed Bluetooth, WLAN, iAP2 and AirPlay evidence belongs in the subsystem documents. This document connects those facts without promoting unresolved hypotheses into implementation requirements.
