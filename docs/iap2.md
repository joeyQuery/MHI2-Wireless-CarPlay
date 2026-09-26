# MHI2 iAP / iAP2 Architecture

Evidence-driven map of the iAP/iAP2 infrastructure in MHI2, focused on the transport boundary required for Wireless CarPlay.

> **Status:** Active research  
> **Platform:** Audi MHI2 / QNX  
> **Target:** Wireless CarPlay

---

# 1. Current Mapped Architecture

## 1.1 Production USB iAP2 path

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    USB["USB"]
    IPOD["/dev/ipod0"]
    DRIVER["iAP2 driver"]
    DIO["dio_manager"]
    AIRPLAY["libairplay.so"]

    PHONE --> USB
    USB --> IPOD
    IPOD --> DRIVER
    DRIVER --> DIO
    DIO --> AIRPLAY
~~~

The production DIO/iAP2 path is tied to the USB-derived /dev/ipod0 boundary.

## 1.2 Discovered iAP2 infrastructure

The firmware contains:

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

These establish iAP/iAP2 infrastructure in the image. They do not prove that Wireless CarPlay iAP2 is enabled.

## 1.3 Bluetooth-side iAP

Production Bluetooth configuration contains:

~~~text
enableIap=false
~~~

while the image contains:

~~~text
libasimmxconnectivity_bluetooth_iapproxy.so
~~~

~~~mermaid
flowchart LR
    CONFIG["enableIap=false"]
    BLUETOOTH["bluetooth"]
    PROXY["Bluetooth iAP proxy"]
    IAP2["iAP2"]

    CONFIG -.-> BLUETOOTH
    BLUETOOTH --> PROXY
    PROXY --> IAP2
~~~

The exact branch controlled by enableIap remains unresolved.

## 1.4 DIO CarPlay boundary

Relevant DIO symbols include:

~~~text
notifyiAP2DeviceConnected
notifyiAP2DeviceDisconnected
iAP2Connect
requestCarPlay
checkCarPlayCompatibility
~~~

~~~mermaid
flowchart LR
    IAP2["iAP2"]
    EVENT["DIO iAP2 event"]
    COMPAT["checkCarPlayCompatibility"]
    REQUEST["requestCarPlay"]
    SESSION["CarPlay session"]

    IAP2 --> EVENT
    EVENT --> COMPAT
    COMPAT --> REQUEST
    REQUEST --> SESSION
~~~

This is the key iAP2-to-CarPlay boundary currently visible in DIO.

---

# 2. What Remains to Be Traced

## 2.1 enableIap

Highest-priority target:

~~~text
bluetooth
  ↓
configuration parser
  ↓
enableIap
  ↓
unknown registration/startup path
  ↓
Bluetooth iAP proxy
~~~

Determine exactly what production disables with enableIap=false.

## 2.2 DIO iAP2 transport boundary

The central question is whether CIpodAP2Service is fundamentally:

- a USB device service; or
- a transport-neutral iAP2 service with a USB-specific configuration.

Trace:

~~~mermaid
flowchart LR
    DIO["CIpodAP2Service"]
    DEVICE["/dev/ipod0"]
    CONNECT["iAP2 connect"]
    CLIENT["libiap2client"]
    TRANSPORT["Unknown transport abstraction"]

    DIO --> DEVICE
    DIO --> CONNECT
    CONNECT --> CLIENT
    CLIENT -.-> TRANSPORT
~~~

## 2.3 iAP2 NCM

The image contains:

~~~text
devu-iap2ncm-tegra3-ci.so
~~~

This is relevant because NCM is a network transport mechanism.

The distinction must be preserved:

~~~text
iAP2 protocol
    ≠
USB NCM transport
~~~

The component's exact production role remains unresolved.

## 2.4 Bluetooth iAP transport

Target:

~~~mermaid
flowchart LR
    PHONE["iPhone"]
    BT["Bluetooth"]
    IAP["iAP2"]
    DIO["dio_manager"]

    PHONE <--> BT
    BT <--> IAP
    IAP --> DIO
~~~

The exact Bluetooth protocol/channel used by the production image remains to be traced.

## 2.5 iAP2 → CarPlay

The required trace is:

~~~mermaid
flowchart LR
    IAP2["iAP2"]
    EVENT["notifyiAP2DeviceConnected"]
    COMPAT["CarPlay compatibility"]
    REQUEST["requestCarPlay"]
    SESSION["CarPlay session"]

    IAP2 --> EVENT
    EVENT --> COMPAT
    COMPAT --> REQUEST
    REQUEST --> SESSION
~~~

Recover exact callbacks, arguments and state transitions.

---

# 3. Component / Subsystem Breakdown

## 3.1 libiap2client.so

Trace for:

- transport creation;
- device opening;
- channel/session creation;
- packet framing;
- callbacks;
- service registration;
- disconnect handling.

No transport semantics should be assigned without binary evidence.

## 3.2 ipod-drvr-iap2.so

Production USB/iPod-side iAP2 component.

Correlate it with:

~~~text
/dev/ipod0
CIpodAP2Service
mss-ipodiap2.so
~~~

## 3.3 mss-ipodiap2.so

Another production iAP2-related component. Ownership relative to ipod-drvr-iap2.so and DIO remains to be fully traced.

## 3.4 devu-iap2-tegra3-ci.so

iAP2 device/transport component in the firmware. Exact production role remains to be traced.

## 3.5 devu-iap2ncm-tegra3-ci.so

NCM-related iAP2 component. Trace alongside:

~~~text
carplay0
devnp-usbdnet.so
/dev/ipod0
~~~

to establish whether these are independent or linked parts of the production path.

## 3.6 /eso/bin/apps/iap

The iAP application/process exists.

Remaining:

- who launches it;
- what transport it opens;
- whether it is active in production;
- how it communicates with DIO;
- whether it is used for Bluetooth iAP.

## 3.7 Bluetooth iAP proxy

~~~text
libasimmxconnectivity_bluetooth_iapproxy.so
~~~

This is the key candidate for Bluetooth-side iAP integration.

Trace its callers, exports, interfaces and registration path before modifying code.

---

# 4. Evidence Status

## Proven

- iAP/iAP2 infrastructure exists in the production image.
- Production CarPlay uses an iAP2 path involving /dev/ipod0.
- DIO contains explicit iAP2/CarPlay state-machine methods.
- A Bluetooth iAP proxy library exists.
- Production Bluetooth configuration has enableIap=false.
- iAP2 NCM and other iAP2 components exist.

## Partially traced

- DIO → iAP2 service;
- CIpodAP2Service transport boundary;
- libiap2client.so role;
- iAP2 NCM role;
- Bluetooth iAP proxy ownership.

## Not yet proven

- that enabling enableIap produces a usable Wireless CarPlay iAP2 path;
- that Bluetooth iAP is the exact Wireless CarPlay bootstrap;
- that DIO can consume wireless iAP2 without modification;
- that devu-iap2ncm-tegra3-ci.so is used by production CarPlay;
- that /dev/ipod0 is merely configuration rather than hard-coded transport logic.

---

# 5. End State / Trace Objective

~~~mermaid
sequenceDiagram
    participant P as iPhone
    participant B as Bluetooth
    participant I as iAP2
    participant D as dio_manager
    participant A as libairplay

    P->>B: CarPlay bootstrap
    B->>I: iAP2 transport
    I->>D: iAP2 device/session
    D->>D: compatibility / requestCarPlay
    D->>A: AirPlay receiver session
    A-->>P: CarPlay session
~~~

For every transition identify the exact process, library, IPC, device/socket, function, callback and protocol message.

---

# 6. Wireless CarPlay Integration Boundary

Current:

~~~text
iPhone
  ↓ USB
/dev/ipod0
  ↓
iAP2
  ↓
DIO
~~~

Target:

~~~text
iPhone
  ↓ Bluetooth bootstrap
wireless iAP2
  ↓
DIO
~~~

The network transport is parallel:

~~~text
iPhone
  ↓ Wi-Fi
uap0
  ↓
mDNS / AirPlay
  ↓
DIO / libairplay
~~~

The two paths must correlate into one CarPlay session.

---

# 7. Highest-Value Next Traces

1. Trace enableIap from configuration parser to branch.
2. Trace callers of the Bluetooth iAP proxy.
3. Trace libiap2client.so constructors and transport creation.
4. Trace CIpodAP2Service device opening and callbacks.
5. Trace devu-iap2ncm-tegra3-ci.so.
6. Correlate iAP2 events with DIO notifyiAP2DeviceConnected.
7. Determine whether Bluetooth iAP can coexist with the existing Wi-Fi/AP path.
8. Capture a complete Bluetooth → iAP2 → DIO sequence before changing transport code.

---

# 8. Evidence Discipline

Presence of an iAP2 binary proves infrastructure exists, not that a production Wireless CarPlay path is enabled.

The final goal is a function-level transport map rather than a component inventory.
