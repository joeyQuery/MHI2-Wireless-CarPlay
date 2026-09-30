# MHI2 Wireless CarPlay

**Plain English** · [**Technical English**](docs/technical-readme.md)

This project is reverse-engineering the production Audi MHI2 firmware to determine how **Wireless CarPlay can be connected to the existing MHI2 CarPlay stack**.

## What we have proved

The firmware already contains the major pieces involved:

- Marvell 8787 Wi-Fi and Bluetooth hardware
- an existing WLAN/AP interface, `uap0`
- Bluetooth iAP/iAP2 infrastructure
- a real Bluetooth iAP transport in `iap`
- DIO's CarPlay integration
- mDNS / Bonjour
- the AirPlay receiver
- the existing USB CarPlay network interface, `carplay0`

The Bluetooth side is now traced far enough to prove that a runtime-supplied endpoint reaches `CIapBTChannel::open64()`. Separately, DIO's current production iAP2 configuration uses `/dev/ipod0`.

## Where we are now

The remaining problem is **not building a Wi-Fi stack**. The hardware and network infrastructure already exist.

The critical unresolved boundary is:

```text
Bluetooth iAP
    ↓
runtime endpoint
    ↓
? same iAP2 resource-manager path ?
    ↓
DIO / CarPlay
```

On the network side, we also still need to trace the actual AirPlay packet/multicast interface selection. The exported screen interface setters in this MU0678 build are no-op stubs, so they are not being treated as the selector.

**Wireless CarPlay is not yet proven or working end-to-end.**

The repository records proven findings, partial traces, disproven interpretations and remaining targets separately.

### Start here

- [Current status](STATUS.md)
- [Roadmap](docs/roadmap.md)
- [Technical documentation](docs/technical-readme.md)
- [Evidence register](docs/evidence.md)
- [Latest trace: Bluetooth iAP2 → DIO / AirPlay boundary](traces/TRACE-013-bluetooth-iap2-dio-airplay-boundary.md)

> **Trace it. Prove it. Document it.**
