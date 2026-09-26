# MHI2 Wireless CarPlay

Reverse engineering MHI2 and CarPlay to help develop a **Wireless Apple CarPlay** on **supported Audi MHI2 units**.

> **Status:** Hiatus on research and development  
> **Platform:** Audi MHI2 on MU0678-class firmware [QNX]

---

# Roadmap

### Understanding

- [x] Map the existing wireless architecture
- [x] Identify the Bluetooth and iAP2 infrastructure
- [x] Identify the Wi-Fi and AirPlay infrastructure
- [ ] Complete the end-to-end Wireless CarPlay trace

### Implementation

- [ ] Establish the wireless connection
- [ ] Connect the wireless transport to MHI2 CarPlay
- [ ] Achieve a working Wireless CarPlay session
- [ ] Validate audio, video and reconnection

### Documentation

- [ ] Complete the architecture documentation
- [ ] Document the final implementation
- [ ] Separate experimental work from the stable implementation

---

# Project

MHI2 systems already contain a substantial wireless connectivity stack, including Bluetooth, iAP/iAP2, Wi-Fi, AirPlay and CarPlay components.

This project investigates how those components work together and how the existing MHI2 architecture can be used to provide Wireless CarPlay.

The focus is on **understanding the platform itself**, rather than replacing it with an unrelated wireless stack.

---

# Architecture

~~~mermaid
flowchart LR
    PHONE["iPhone"]

    subgraph MHI2["Audi MHI2"]
        direction LR
        BT["Bluetooth / iAP2"]
        WIFI["Wi-Fi / uap0"]
        MDNS["mDNS"]
        DIO["dio_manager"]
        AIRPLAY["libairplay"]
        CP["CarPlay"]
        MMI["MMI"]
    end

    PHONE <-->|"Bluetooth"| BT
    PHONE <-->|"Wi-Fi"| WIFI
    WIFI --> MDNS
    BT --> DIO
    MDNS --> DIO
    DIO --> AIRPLAY
    AIRPLAY --> CP
    CP --> MMI
~~~

The detailed integration map is in **[Wireless CarPlay Architecture](docs/wireless-carplay-architecture.md)**.

---

# Evidence

This is an evidence-driven reverse-engineering project.

Conclusions are based on:

- firmware contents
- configuration
- binaries
- runtime behaviour
- controlled experiments
- protocol analysis

Unproven assumptions are explicitly identified rather than presented as fact.

---

# Documentation

Detailed research is maintained in separate documents:

- **[Wireless CarPlay Architecture](docs/wireless-carplay-architecture.md)** — end-to-end integration map, evidence status and trace objectives
- **[Bluetooth](docs/bluetooth.md)** — Bluetooth architecture and behaviour
- **[iAP2](docs/iap2.md)** — iAP/iAP2 architecture and transport
- **[Wi-Fi](docs/wifi.md)** — MHI2 WLAN architecture
- **[AirPlay](docs/airplay.md)** — AirPlay and mDNS
- **[CarPlay](docs/carplay.md)** — MHI2 CarPlay integration
- **[Wireless Capability Breakdown](docs/wireless-capability-breakdown.md)** — broader wireless/connectivity findings

---

# Primary Reference

**[LIVI](https://github.com/f-io/LIVI)** is the primary open-source reference implementation used for Wireless CarPlay protocol research.

LIVI is used as a reference, not as evidence of MHI2's internal implementation.

---

# Safety

This project involves reverse engineering and modifying automotive infotainment firmware.

Always keep original files, hashes and a reliable recovery path before experimenting with a vehicle.

---

> **Trace it. Prove it. Document it.**
