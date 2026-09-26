# MHI2 Wireless CarPlay Glossary

Terminology used throughout the repository.

## AirPlay
Apple's media/session protocol layer used by the MHI2 CarPlay receiver. In this project, `libairplay.so` contains receiver/session and Bonjour/mDNS-related functionality.

## Bonjour / mDNS
Service discovery and advertisement mechanisms used by AirPlay/CarPlay networking. The repository has evidence for Bonjour APIs and the `_airplay._tcp.` service.

## CarPlay
The Apple smartphone-integration session/application layer. This project focuses on connecting wireless transports to the existing MHI2 CarPlay implementation.

## carplay0
The production USB-derived network interface created through `devnp-usbdnet.so`. It is distinct from `uap0`.

## DIO
The MHI2 integration layer/process represented by `dio_manager`. It contains iAP2, CarPlay session and AirPlay receiver integration points identified in the current investigation.

## iAP / iAP2
Apple accessory communication protocols/components present in the MHI2 firmware. Do not use “iAP2” to mean a specific physical transport; the project is specifically investigating the transport boundary.

## iAP2 NCM
An iAP2-related component/path associated with NCM. Its exact production role remains unresolved. It must not be conflated automatically with the USB network interface `carplay0`.

## NCM
Network Control Model terminology used by USB/network components. In this project, “USB NCM” refers to the USB-derived networking path that produces `carplay0`.

## /dev/ipod0
Production USB-derived device boundary associated with the current iAP2/CarPlay path.

## uap0
The existing Marvell WLAN/AP interface. Known production address:

```text
10.173.189.1/24
```

It is a candidate Wireless CarPlay network interface, not yet proven to be the production AirPlay interface.

## mlan0
The Marvell WLAN station/client interface identified in the broader connectivity architecture. It is distinct from the AP interface `uap0`.

## MDNS_DIRECTLINK_IFACE
Production configuration key whose known value is:

```text
MDNS_DIRECTLINK_IFACE=carplay0
```

Its exact consumer and runtime effect remain unresolved.

## DSI / ASI
MHI2 interface/proxy terminology appearing throughout the connectivity architecture. Individual proxy/stub presence does not by itself establish a complete runtime call graph.

## Transport
The underlying communication mechanism used by a protocol/service. A protocol such as iAP2 should not automatically be treated as synonymous with USB, Bluetooth or NCM.

## Bootstrap
The initial device/session establishment phase preceding the complete CarPlay media session. The exact MHI2 Bluetooth → iAP2 → Wi-Fi correlation remains an open trace.

## Session
A logical CarPlay/AirPlay interaction associated with a phone. Session creation, mode changes and finalization are exposed by DIO/AirPlay integration symbols.

## Screen / Stream
A CarPlay media/display path. Use these terms only for the media path actually being discussed; do not assign protocol stream identifiers unless the relevant evidence is part of the current Wireless CarPlay investigation.

## Evidence Classes
See [Evidence](evidence.md): Static evidence, Runtime evidence, Protocol evidence, Controlled experiment and Inference.
