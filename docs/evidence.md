# MHI2 Evidence Register

This is the central evidence ledger for the project. It prevents architectural diagrams, repeated prose and hypotheses from becoming indistinguishable from verified MHI2 behaviour.

## Evidence Classes

- **Static** — firmware strings, symbols, configuration, extracted files, disassembly and binary structure.
- **Runtime** — observed processes, interfaces, state transitions, commands, logs and system behaviour on the MHI2.
- **Protocol** — HCI, iAP2, mDNS, AirPlay or other protocol-level captures.
- **Controlled Experiment** — deliberate modification or controlled action followed by observation.
- **Inference** — technical interpretation that still requires MHI2-specific confirmation.

## Status

- **Proven** — directly supported by MHI2-specific evidence.
- **Partially traced** — relevant components/boundary are established but the complete path is missing.
- **Not yet proven** — plausible interpretation without sufficient MHI2-specific proof.
- **Disproven** — investigation has produced evidence against the previous interpretation.

## Register

| ID | Finding | Class | Status | Source / location | Notes |
|---|---|---|---|---|---|
| E-001 | Marvell 8787 provides the WLAN/BT platform | Static | Proven | `docs/wifi.md`, `docs/bluetooth.md` | Shared controller established |
| E-002 | `uap0` exists as the AP interface | Static + runtime | Proven | `docs/wifi.md` | Known address `10.173.189.1/24` |
| E-003 | `carplay0` is the USB-derived CarPlay network interface | Static + runtime | Proven | architecture docs | Created through `devnp-usbdnet.so` |
| E-004 | Production configuration contains `MDNS_DIRECTLINK_IFACE=carplay0` | Static | Proven | AirPlay/architecture docs | Consumer remains unresolved |
| E-005 | Production CarPlay uses `/dev/ipod0` | Static + runtime | Proven | `docs/iap2.md` | Exact transport abstraction remains unresolved |
| E-006 | `enableIap=false` is present in production Bluetooth configuration | Static | Proven | `docs/bluetooth.md`, `docs/iap2.md` | Exact branch remains unresolved |
| E-007 | Bluetooth-side iAP proxy exists | Static | Proven | `libasimmxconnectivity_bluetooth_iapproxy.so` | Does not prove Wireless CarPlay use |
| E-008 | AirPlay exposes Bonjour/mDNS APIs | Static | Proven | `libairplay.so` | Includes `_airplay._tcp.` |
| E-009 | AirPlay exposes interface-selection helpers | Static | Proven | `libairplay.so` | Wi-Fi/USB-aware helpers present |
| E-010 | DIO contains iAP2 and AirPlay/CarPlay integration symbols | Static | Proven | `dio_manager` | Exact call arguments still require tracing |
| E-011 | WLAN/Bluetooth coexistence configuration exists | Static | Proven | `coex.cfg` / Wi-Fi docs | Does not prove CarPlay-specific tuning |
| E-012 | `111` is the alternate AirPlay stream type | Protocol / controlled investigation | Proven | Existing AltScreen investigation | It does not itself contain display description |
| E-013 | `altScreen` is AirPlay feature negotiation rather than iAP2 Identify | Protocol / static | Proven | Existing AltScreen investigation | Keep separate from iAP2 capability negotiation |
| E-014 | `hci0` / `/dev/ttyS0` lab configuration is not production proof | Static comparison | Proven as limitation | `docs/bluetooth.md` | Do not use it as production HCI transport |
| E-015 | Wireless CarPlay is not yet proven end-to-end | Cross-system | Proven | Current repository state | Target remains an investigation target |

## Adding Entries

Every significant new finding should answer:

1. What exactly was observed?
2. Which evidence class supports it?
3. Which firmware/binary/configuration produced it?
4. What is proven versus inferred?
5. Can another researcher reproduce it?
6. Does it contradict an existing entry?

Use a stable new ID rather than silently rewriting an existing finding when evidence changes. Mark old interpretations as superseded/disproven where appropriate.

## Source Discipline

Subsystem documents may explain findings in detail, but this register is the cross-project index of what the project currently treats as evidence. Repetition across documents does not upgrade an inference to proven status.
