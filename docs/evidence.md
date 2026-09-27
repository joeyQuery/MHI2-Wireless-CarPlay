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
| E-003 | `carplay0` is the USB-derived CarPlay network interface | Static + runtime | Proven | `docs/wifi.md`, `docs/carplay.md`, TRACE-003 | Created through `devnp-usbdnet.so`; `mdnsd` has a proven direct-link consumer, while boot-time environment provenance and final socket binding remain unresolved |
| E-004 | Production configuration contains `MDNS_DIRECTLINK_IFACE=carplay0` | Static | Proven | `docs/airplay.md`, `docs/wireless-carplay-architecture.md`, TRACE-003 | Configuration value is proven; `mdnsd` consumption is now proven, but boot-time environment propagation and final socket binding remain unresolved |
| E-005 | Production CarPlay uses `/dev/ipod0` | Static + runtime | Proven | `docs/iap2.md`, TRACE-002 | Exact service/device-open path and transport abstraction remain unresolved |
| E-006 | `enableIap=false` is present in production Bluetooth configuration | Static | Proven | `docs/bluetooth.md`, `docs/iap2.md` | Exact branch remains unresolved |
| E-007 | Bluetooth-side iAP proxy exists | Static | Proven | `libasimmxconnectivity_bluetooth_iapproxy.so` | Does not prove Wireless CarPlay use |
| E-008 | AirPlay exposes Bonjour/mDNS APIs | Static | Proven | `libairplay.so` | Includes `_airplay._tcp.` |
| E-009 | AirPlay exposes interface-selection helpers | Static | Proven | `libairplay.so` | Wi-Fi/USB-aware helpers present |
| E-010 | DIO contains iAP2 and AirPlay/CarPlay integration symbols | Static | Proven | `dio_manager` | Exact call arguments still require tracing |
| E-011 | WLAN/Bluetooth coexistence configuration exists | Static | Proven | `coex.cfg` / Wi-Fi docs | Does not prove CarPlay-specific tuning |
| E-012 | `hci0` / `/dev/ttyS0` lab configuration is not production proof | Static comparison | Proven as limitation | `docs/bluetooth.md` | Do not use it as production HCI transport |
| E-013 | Wireless CarPlay is not yet proven end-to-end | Cross-system | Proven | Current repository state | Target remains an investigation target |
| E-014 | Production `iap` contains a dedicated Bluetooth iAP channel implementation | Static | Proven | `eso/bin/apps/iap` | `CIapBTChannel` implements open/connect/read/write/close/update; `openiAPDevice` reaches `open64`; no `/dev/ipod0` literal was found in this binary |
| E-015 | Bluetooth iAP uses an IPC-facing IapDeviceServices boundary | Static | Proven | `eso/bin/apps/iap`, `eso/lib/factories/libasimmxconnectivity_bluetooth_iapproxy.so` | Proxy/service/reply classes identify `asi.connectivity.bluetooth.iap.IapDeviceServices`; exact endpoint publication/consumer path remains unresolved |
| E-016 | DIO's production iAP2 boundary is explicitly `/dev/ipod0` | Static | Proven | `eso/bin/apps/dio_manager` | `iap2.device` resolves to `/dev/ipod0`; `iap2_connect`/`iap2_disconnect` are dynamically referenced with direct callsites |
| E-017 | DIO contains production `MDNS_DIRECTLINK_IFACE=carplay0` configuration | Static | Proven | `eso/bin/apps/dio_manager` | String is present in configuration metadata; `mdnsd` consumption is proven independently; DIO/config → running `mdnsd` environment propagation remains unresolved |
| E-018 | Production AirPlay screen interface setters are no-op stubs | Static / Disassembly | Proven | pristine `eso/lib/libairplay.so` | `SetClientIfMACAddr`, `SetIFName`, and `SetTransportType` each reduce to `bx lr`; their presence does not prove active interface selection |
| E-019 | `SocketSetBoundInterface` is not an active selector in production `libairplay.so` | Static / Disassembly | Proven | pristine `eso/lib/libairplay.so` | Entry is a tiny stub/constant-return; do not treat it as the production binding mechanism |
| E-020 | AirPlay still contains real packet/multicast interface and Wi-Fi classification helpers | Static / Disassembly | Proven | pristine `eso/lib/libairplay.so` | `SocketSetPacketReceiveInterface`, `SocketSetMulticastInterface`, and `IsWiFiNetworkInterface` have substantive implementations; actual production call path remains unresolved |
| E-021 | Bluetooth active-device callback reaches `CIapBTChannel::updateiAPDevice()` | Static / Disassembly | Proven | `eso/bin/apps/iap` | `CIapDeviceServicesReplyImpl::updateActiveDevices()` extracts an active-device object and supplies its endpoint/device string to the Bluetooth iAP channel |
| E-022 | Bluetooth iAP endpoint is runtime-supplied and reaches `open64()` | Static / Disassembly | Proven | `eso/bin/apps/iap` | `updateiAPDevice(...)` stores the supplied `CIString`; `openiAPDevice()` converts it to a native path and calls `open64()` |
| E-023 | Bluetooth iAP endpoint must not be equated with DIO `/dev/ipod0` | Static / Disassembly | Proven as separation | `eso/bin/apps/iap`, `eso/bin/apps/dio_manager` | `iap` contains no `/dev/ipod0` literal while DIO independently configures `/dev/ipod0`; exact Bluetooth endpoint value remains unresolved |
| E-024 | Production `enableIap=false` gates an otherwise complete Bluetooth iAP/iAP2 implementation | Static | Proven | `eso/production/connectivity.json`, `eso/bin/apps/iap`, Bluetooth iAP proxy | Firmware contains the proxy, active-device callbacks, Bluetooth channel, iAP2 packet/link machinery and support checks despite the production configuration disabling IAP |
| E-025 | DIO creates the production AirPlay server through `AirPlayReceiverServerCreate(...)` | Static / Disassembly | Proven | `dio_manager` | Production path uses the direct server-creation API rather than the config-file creation variant |
| E-026 | AirPlay Bonjour registration uses its `interfaceName` field as an interface index | Static / Disassembly | Proven | pristine `eso/lib/libairplay.so` | `_UpdateBonjourAirPlay` reads `interfaceName` at object+0x6c, calls `if_nametoindex()` when non-empty, and passes the resulting index to `DNSServiceRegister()` |
| E-027 | AirPlay screen interface/transport/client-MAC setters are production no-ops | Static / Disassembly | Proven | pristine `eso/lib/libairplay.so` | `AirPlayReceiverSessionScreen_SetIFName`, `SetTransportType`, and `SetClientIfMACAddr` reduce to `bx lr` |
| E-028 | AirPlay DNS-SD calls form the boundary into the system mDNS implementation | Static | Proven | `libairplay.so`, `libdns_sd.so`, `mdnsd` | AirPlay calls `DNSServiceRegister`, `DNSServiceUpdateRecord`, `DNSServiceQueryRecord`, and `DNSServiceGetAddrInfo`; the system contains the corresponding DNS-SD/mDNS implementation |
| E-029 | `mdnsd` explicitly consumes `MDNS_DIRECTLINK_IFACE` | Static / Disassembly | Proven | `mdnsd` | `SetupOneInterface()` constructs the variable name and calls `getenv("MDNS_DIRECTLINK_IFACE")` |
| E-030 | `mdnsd` has dedicated direct-link interface registration handling | Static / Disassembly | Proven | `mdnsd` | `SetupInterfaceList()` feeds interfaces into `SetupOneInterface()`, which retains the interface name and registers the interface with the mDNS platform |
| E-031 | Production boot-time export of `MDNS_DIRECTLINK_IFACE=carplay0` into `mdnsd` remains unproven | Static limitation | Not yet proven | `dio_manager` configuration metadata + `mdnsd` | The value exists in production configuration, and `mdnsd` consumes the environment variable, but no recovered boot-time `putenv()`/environment-propagation edge proves the running daemon receives it |

## Trace Cross-Reference

The transport-boundary traces are maintained under [`traces/`](../traces/README.md). A trace may only strengthen an evidence entry when its underlying observation is actually recovered; the trace status itself is not evidence.

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

Subsystem documents may explain findings in detail, but this register is the cross-project index of what the project currently treats as evidence. Repetition across documents does not upgrade an inference to proven status. For forensic findings, record the actual binary, configuration or trace artifact rather than citing only an architecture summary. If runtime evidence is not reproducible from a committed artifact, say so explicitly instead of presenting the observation as repository-reproducible.
