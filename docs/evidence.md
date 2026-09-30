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
| E-002 | `uap0` exists as the AP interface | Static + runtime observation | Proven | `docs/wifi.md` | Known address `10.173.189.1/24`; runtime state was observed on the MHI2, but no standalone runtime capture artifact is committed in this repository |
| E-003 | `carplay0` is the USB-derived CarPlay network interface | Static + runtime observation | Proven | `docs/wifi.md`, `docs/carplay.md`, TRACE-003 | Static creation path is repository-reproducible; runtime presence was observed on the MHI2, but no standalone runtime capture artifact is committed. `mdnsd` has a proven direct-link consumer, while boot-time environment provenance and final socket binding remain unresolved |
| E-004 | Production configuration contains `MDNS_DIRECTLINK_IFACE=carplay0` | Static | Proven | `docs/airplay.md`, `docs/wireless-carplay-architecture.md`, TRACE-003 | Configuration value is proven; `mdnsd` consumption is now proven, but boot-time environment propagation and final socket binding remain unresolved |
| E-005 | Production CarPlay uses `/dev/ipod0` | Static + runtime observation | Proven | `docs/iap2.md`, TRACE-002 | The production configuration/static boundary is repository-reproducible; runtime use was observed on the MHI2, but no standalone runtime capture artifact is committed. Exact service/device-open path and transport abstraction remain unresolved |
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
| E-024 | Production configuration contains `enableIap=false` while substantial Bluetooth iAP/iAP2 implementation exists in the image | Static | Proven | `eso/production/connectivity.json`, `eso/bin/apps/iap`, Bluetooth iAP proxy | The configuration value and implementation components are independently proven. The exact `enableIap` parser/control-flow branch is unresolved, so this entry does **not** claim that `enableIap=false` is the activation gate or that changing it enables Wireless CarPlay |
| E-025 | DIO creates the production AirPlay server through `AirPlayReceiverServerCreate(...)` | Static / Disassembly | Proven | `dio_manager` | Production path uses the direct server-creation API rather than the config-file creation variant |
| E-026 | AirPlay Bonjour registration uses its `interfaceName` field as an interface index | Static / Disassembly | Proven | pristine `eso/lib/libairplay.so` | `_UpdateBonjourAirPlay` reads `interfaceName` at object+0x6c, calls `if_nametoindex()` when non-empty, and passes the resulting index to `DNSServiceRegister()` |
| E-027 | AirPlay screen interface/transport/client-MAC setters are production no-ops | Static / Disassembly | Proven | pristine `eso/lib/libairplay.so` | `AirPlayReceiverSessionScreen_SetIFName`, `SetTransportType`, and `SetClientIfMACAddr` reduce to `bx lr` |
| E-028 | AirPlay DNS-SD calls form the boundary into the system mDNS implementation | Static | Proven | `libairplay.so`, `libdns_sd.so`, `mdnsd` | AirPlay calls `DNSServiceRegister`, `DNSServiceUpdateRecord`, `DNSServiceQueryRecord`, and `DNSServiceGetAddrInfo`; the system contains the corresponding DNS-SD/mDNS implementation |
| E-029 | `mdnsd` explicitly consumes `MDNS_DIRECTLINK_IFACE` | Static / Disassembly | Proven | `mdnsd` | `SetupOneInterface()` constructs the variable name and calls `getenv("MDNS_DIRECTLINK_IFACE")` |
| E-030 | `mdnsd` has dedicated direct-link interface registration handling | Static / Disassembly | Proven | `mdnsd` | `SetupInterfaceList()` feeds interfaces into `SetupOneInterface()`, which retains the interface name and registers the interface with the mDNS platform |
| E-031 | Production boot-time export of `MDNS_DIRECTLINK_IFACE=carplay0` into `mdnsd` remains unproven | Static limitation | Not yet proven | `dio_manager` configuration metadata + `mdnsd` | The value exists in production configuration, and `mdnsd` consumes the environment variable, but no recovered boot-time `putenv()`/environment-propagation edge proves the running daemon receives it |



| E-032 | MU0678 iAP2 driver contains a generic transport callback layer | Static / Disassembly | Proven | ipod-drvr-iap2.so | link_create() calls transport_get_link_params(); transport_send_pkt(), transport_receive() and transport_recv_pkt() dispatch through transport-owned callback fields |
| E-033 | MU0678 iAP2 driver has separate Bluetooth and Wi-Fi transport-component Identify handlers | Static / Disassembly | Proven | ipod-drvr-iap2.so | ident_info_tspbt() and ident_info_tspwifi() exist, and ident_info_funcs contains entries for both |
| E-034 | Wi-Fi transport component advertises iAP2 and CarPlay capability fields | Static / Data structures | Proven | ipod-drvr-iap2.so | sparams_id_info_wifitspcomp contains TransportComponentName, TransportSupportsiAP2Connection, and TransportSupportsCarPlay descriptors |
| E-035 | Bluetooth transport component advertises iAP2 and Bluetooth MAC-related fields | Static / Data structures | Proven | ipod-drvr-iap2.so | sparams_id_info_tspbt contains TransportComponentName, TransportSupportsiAP2Connection, and BluetoothTransportMediaAccessControlAddress descriptors |
| E-036 | Shipped MU0678 iAP2 configuration selects a Lightning/USB transport; its Bluetooth connection-status section is commented out | Static / Configuration | Proven | /etc/mm/iap2.cfg | [transport] is name=Lightning Connector. The separate [bluetooth] section is explicitly documented as Bluetooth Connection Status handling, with enable/id/name/connect-status/mac fields commented out. This does not by itself disable or select the underlying iAP2 transport. See E-052 |
| E-037 | Stock smartphone integration monitors /dev/ipod0 and launches DIO as its CarPlay child | Static / Configuration | Proven | smartphone_integrator.json | paths.mcdMonitored=[/dev/ipod0]; child carplay executes dio_manager; shipped smartphone orchestration is USB-device-driven |
| E-038 | The production mDNS preparation script does not export MDNS_DIRECTLINK_IFACE | Static / Script | Proven negative | /etc/scripts/mdnsd.sh | The script only recreates/chmods /var/run/mdnsd; no environment assignment or export is present |

| E-044 | MU0678 ships an explicit USB iAP2-NCM CarPlay device descriptor | Static / Configuration | Proven | `usblauncher_carplay_descriptor.lua` | Product is `iAP2 NCM Accessory`; descriptor combines a vendor-specific iAP interface with CDC Ethernet/NCM control and data interfaces. This establishes an explicit USB iAP2-NCM architecture; it does not prove the NCM component cannot be reused by another transport |


| E-045 | MU0678 libiap2client.so exposes iAP2 client APIs but no recovered Bluetooth/Wi-Fi transport selector | Static / ELF imports/exports | Proven | armle/usr/lib/libiap2client.so | Exports iap2_connect/disconnect and related iAP2 APIs; imports MsgSend/MsgSendv/MsgSendsv/MsgSendvs plus open/close/ionotify; no Bluetooth, Wi-Fi, HCI or socket transport symbols were recovered |
| E-046 | MU0678 mss-ipodiap2.so exposes the same core iAP2 client API surface | Static / ELF exports | Proven | armle/lib/dll/mss-ipodiap2.so | Exports iap2_connect/disconnect and related iAP2 APIs and identifies itself as an iPod iAP2 Media Synchronizer; exact caller/ownership boundary remains unresolved |
| E-047 | MU0678 devu-iap2ncm-tegra3-ci.so is a ChipIdea USB device-controller implementation for an iAP2 NCM accessory | Static / ELF strings/exports | Proven | armle/lib/dll/devu-iap2ncm-tegra3-ci.so | Contains the ChipIdea USB controller implementation, io-usb-dcd -diap2ncm-tegra3-ci and an explicit iAP2 NCM accessory description; this does not establish wireless reuse |
| E-048 | iap2cli uses /dev/ipod0 as its default iPod mountpoint while linking against libiap2client | Static / Strings/imports | Proven | armle/usr/bin/iap2cli | Help text specifies default /dev/ipod0 and the binary imports iap2_connect from libiap2client; this does not prove every iap2_connect call is hard-coded to /dev/ipod0 |

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

| E-039 | MU0678 iAP2 driver implements executable Bluetooth/Wi-Fi feature-startup machinery | Static / Disassembly | Proven | ipod-drvr-iap2.so | `iap2_start_features()` reaches Bluetooth feature handlers including `bt_send_info()` and `bt_start_updates()`; this is compiled capability, not proof of production activation |
| E-040 | MU0678 iAP2 packet dispatcher handles Bluetooth and accessory-Wi-Fi control events | Static / Disassembly | Proven | ipod-drvr-iap2.so | `link_handle_iap2pkt()` directly reaches `bt_update_recv()` and `wifi_acc_config_info()`; recovered targets include 0x13818 and 0x1382c |
| E-041 | MU0678 iAP2 contains an accessory-Wi-Fi configuration response path | Static / Disassembly | Proven | ipod-drvr-iap2.so | Handles RequestAccessoryWiFiConfigurationInformation and returns SSID, passphrase, security type and channel; WPA2 configuration/error handling is present |
| E-042 | DIO contains a dedicated Bluetooth smartphone integration boundary with explicit CarPlay mode handling | Static | Proven | eso/bin/apps/dio_manager | `BluetoothSmartphoneIntegration`, `BluetoothSmartphoneIntegrationReply`, `CBluetoothController` and CarPlay mode/state operations are present; packet-level Bluetooth iAP2 → DIO handoff remains unresolved |
| E-043 | DIO contains mDNS environment-setting machinery and a producer lead | Static / Disassembly | Proven as capability, not propagation | eso/bin/apps/dio_manager | `setEnv`, `getEnvName`, `putenv(%s): %s`, `startMdnsdProcess` and `MDNS_DIRECTLINK_IFACE=carplay0` are present; the exact putenv-to-mdnsd edge is not recovered and E-031 remains unresolved |


| E-049 | MU0678 libiap2client.so::iap2_connect() is path-driven and opens its caller-supplied first argument | Static / Disassembly | Proven | armle/usr/lib/libiap2client.so | At 0x2cec the first argument is forwarded to the open PLT entry at 0x114c; the returned descriptor is stored and a 20-byte iAP2 control message is sent with MsgSend. No /dev/ipod0 literal is required by the ABI |
| E-050 | DIO CIpodAP2Service reaches imported iap2_connect through helper 0x15ddb4 with a caller-supplied path | Static / Disassembly | Proven | eso/bin/apps/dio_manager | _ZTVN3dio15CIpodAP2ServiceE at 0x19bc58 contains target 0x15e240; 0x15e2f4 passes r9 to helper 0x15ddb4, which calls iap2_connect at 0x115de8 and stores the returned handle |
| E-051 | Production /dev/ipod0 is a configured path, not an intrinsic iap2_connect() transport restriction | Static / Disassembly + Configuration | Proven | dio_manager, production dio_manager.json, libiap2client.so | Current DIO configuration supplies /dev/ipod0, while the recovered caller/ABI passes a runtime path into iap2_connect; an alternate QNX resource-manager endpoint is therefore structurally possible at this boundary, but Bluetooth substitution is not yet proven |
| E-052 | The `[bluetooth]` section in MU0678 `/etc/mm/iap2.cfg` is a Bluetooth connection-status feature, not the iAP2 transport selector | Static / Configuration | Proven | `/etc/mm/iap2.cfg` | The file's own comments state that `[transport]` describes the transport to the Apple device, while `[bluetooth]` handles Bluetooth Connection Status and uses `IAP2_BT_STATUS_MAC_ADDR` first. Therefore the commented `[bluetooth]` block cannot be used as evidence that Bluetooth iAP2 transport is disabled. The underlying transport-selection/activation path remains unresolved |

| E-053 | MU0678 ipod-drvr-iap2.so exposes a resource-manager driver object whose handler table includes iap2_init(), iap2_msg(), iap2_notify(), iap2_unblock(), OCB lifecycle handlers and fsys_open/read/lseek | Static / ELF data + symbols | Proven | armle/lib/dll/ipod-drvr-iap2.so | ipod_module at 0x27e7c points to iap2_drvr at 0xa9bb0; iap2_drvr+0x18=iAP2 init, +0x24=iAP2 message handler, +0x3c/+0x40/+0x44=filesystem handlers. |
| E-054 | libiap2client.so::iap2_connect() and ipod-drvr-iap2.so::iap2_msg() share an exact QNX message ABI marker | Static / Disassembly | Proven | armle/usr/lib/libiap2client.so; armle/lib/dll/ipod-drvr-iap2.so | iap2_connect opens the caller-supplied path, sends a 20-byte MsgSend message with halfword 0x9999 at offset +6, and expects a 4-byte reply; iap2_msg checks the same +6 field and replies with value 2 via MsgReply. |
| E-055 | ipod-drvr-iap2.so mounts a runtime endpoint through ipod_resmgr_mount() during iAP2 feature startup | Static / Disassembly | Proven | armle/lib/dll/ipod-drvr-iap2.so | iap2_start_features() calls ipod_resmgr_mount(r9, 1), where r9 is loaded from the iAP2 context at +0x14; the runtime mountpoint value remains unresolved. |


| E-056 | TRACE-013 consolidates the production Bluetooth iAP2 → DIO/AirPlay boundary | Static / Disassembly | Proven as trace state | TRACE-013 | The trace establishes the production Bluetooth iAP2 components and separates proven edges from the unresolved endpoint/DIO handoff. |
| E-057 | Production `iap::CIapBTChannel` is wired into the iAP connector state machine and performs open/read/write/close operations | Static / Disassembly | Proven | `eso/bin/apps/iap` | Bluetooth iAP is an active production subsystem, not merely a pairing/configuration artifact. |
| E-058 | Bluetooth iAP uses the `asi.connectivity.bluetooth.iap.IapDeviceServices` RPC boundary | Static / Binary structure | Proven | `iap`, `libasimmxconnectivity_bluetooth_iapproxy.so` | The service/proxy/reply machinery is present; exact endpoint publication remains unresolved. |
| E-059 | Bluetooth iAP's endpoint is runtime-supplied to `open64()` and must not be equated with DIO's configured `/dev/ipod0` without further evidence | Static / Disassembly | Proven separation | `iap`, `dio_manager` | `iap` contains no recovered `/dev/ipod0` literal; DIO independently supplies `/dev/ipod0` to `iap2_connect()`. |
| E-060 | Production AirPlay screen interface/transport/client-MAC setters are not active transport selectors | Static / Disassembly | Proven correction | pristine `libairplay.so` | `SetIFName`, `SetTransportType`, and `SetClientIfMACAddr` are no-op stubs in MU0678. |
| E-061 | The remaining static Bluetooth blocker is inside the production `bluetooth` executable or a runtime endpoint capture | Static limitation | Proven limitation | TRACE-013 | The endpoint value/owner is not recoverable from the currently accessible `iap` and DIO binaries; do not invent the missing branch. |

| E-060 | MU0678 launches btstack as an independently supervised production process | Static / Configuration | Proven | MU0678 `eso/production/connectivity.json` | `btstack` has its own application entry, executable path `/eso/bin/apps/btstack`, and precondition `/tmp/mvloaded`; it is not started by the Bluetooth app's `enableIap` setting. |
| E-061 | `bluetooth.enableIap=false` is separate from btstack process activation | Static / Configuration | Proven separation | MU0678 `eso/production/connectivity.json` | `enableIap` is nested under the `bluetooth` application configuration, while `btstack` has an independent launcher entry. This does not establish what the setting enables/disables internally. |
| E-062 | MU0678 `iap2.cfg` selects Lightning Connector as the iAP2 transport and its Bluetooth section is only documented as Bluetooth Connection Status | Static / Configuration | Proven | MU0678 `mm/iap2.cfg` | `[transport]` is `Lightning Connector`; the commented `[bluetooth]` section documents `enable`, `id`, `name`, `connectstatus`, and `mac` as Bluetooth Connection Status handling. It is not evidence of a transport selector. |
| E-063 | Exact MU0678 `btstack` artifact is present in the dump but its internal ELF contents remain unavailable to the current binary-reading connector | Static / Repository inventory | Proven limitation | MU0678 app-image tree; blob `d9309fc964aa4c6dbe92c180d8ba0dee61899328` | Exact artifact size is 1,912,117 bytes. The connector rejects its non-UTF-8 contents, so no internal btstack endpoint-construction claim is made from this artifact yet. |
| E-064 | Locally inspected MU0678 `bluetooth` binary has Bluetooth/iAP policy strings but no recovered defined iAP endpoint-construction/RFCOMM-server function symbols | Static / ELF inspection | Proven negative evidence | local MU0678 `eso/bin/apps/bluetooth` | Defined symbols include `CBluetoothApplication` and `CBluetoothSmartphoneIntegration`; no defined `CBluetoothIap*`/endpoint-construction/RFCOMM-server functions or direct undefined iAP/RFCOMM endpoint API symbols were recovered. This narrows, but does not mathematically eliminate, ownership by this executable. |
| E-065 | `bluetooth` binary contains `iapEnabled`, `bluetooth.enableIap`, `SERVICETYPE_IAP2`, `ERROR_CARPLAY_ACTIVE`, `Carplay`, and RFCOMM error-handling strings | Static / Strings | Proven | local MU0678 `eso/bin/apps/bluetooth` | These establish policy/service-level integration; they do not establish concrete endpoint creation. |
| E-066 | MU0678 `dio_manager` imports `AirPlayReceiverServerSetProperty`, but its direct caller has not yet been recovered | Static / ELF + relocation/disassembly | Corrected / Partially traced | local pristine MU0678 `dio_manager.stock` | The symbol is a `R_ARM_GLOB_DAT` relocation at `0x19da70), not a normal PLT/JUMP_SLOT entry. The previously claimed direct callsite was actually `AirPlayReceiverServerCreate` at `0x13f0b4` and has been retired. |
| E-067 | MU0678 `dio_manager` does not import the substantive AirPlay socket/interface helper APIs | Static / ELF imports | Proven negative evidence | local pristine MU0678 `dio_manager.stock` | No imports were recovered for `SocketSetPacketReceiveInterface`, `SocketSetMulticastInterface`, `SocketSetBoundInterface`, or `IsWiFiNetworkInterface`; therefore those helpers are not DIO-level direct calls. |

| E-068 | MU0678 production `bluetooth` ELF is now directly inspectable and does not expose a recovered concrete iAP endpoint-construction/RFCOMM-server API | Static / ELF inspection | Proven negative evidence | local MU0678 `eso/bin/apps/bluetooth` | The binary contains `CBluetoothApplication` / `CBluetoothSmartphoneIntegration` and iAP policy/service strings, but no recovered defined `CBluetoothIap*`, endpoint-construction or RFCOMM-server symbols and no direct undefined iAP/RFCOMM endpoint API imports. This narrows ownership; it does not eliminate service-mediated construction. |
| E-069 | MU0678 `bluetooth` contains a real configuration-control function but its `enableIap` field mapping remains unresolved | Static / Disassembly | Partial | local MU0678 `eso/bin/apps/bluetooth` | `CBluetoothApplication::switchBluetoothAccordingToConfig()` at `0x13df18` is called from `diagCBCodingValues(...)` at `0x146500` and checks application-state fields including `0x1182`, `0x1186`, and `0x118a`; no static proof yet ties those fields to `bluetooth.enableIap`. |
