# What MH2p Answers for the MHI2 Open Questions

One place that maps the MHI2 repository's open questions ([STATUS.md](../STATUS.md), the
[roadmap](../docs/roadmap.md), the traces and the evidence register) to what the MH2p firmware shows.

**How to read this.** MH2p is a different unit. Its answers show **how a production VW Group / Alpine
unit solved each problem**. They do not prove what MHI2 (MU0678) does. Each row therefore has two parts:
the MH2p fact (with `P-` evidence) and what it implies for MHI2 (labelled). For each row, the subsystem
document in the last column holds the details.

## STATUS.md open questions

| # | Question (short) | MH2p answer | Implication for MHI2 (inference unless stated) | Details |
| --- | --- | --- | --- | --- |
| 1 | What consumes `enableIap=false`? | MH2p ships `bluetooth.enableIap: true` (`connectivity.json`). The `bluetooth` app reads it into `CBT_CONFIG` next to `global.wirelessCarPlay`; it belongs to `CBluetoothIapManager`'s iAP2 connect/reconnect policy ("iAP2 not allowed. Disconnect"). It does **not** switch on the RFCOMM server, which lives in `btstack` (P-107, P-300). | Read `enableIap` as the `bluetooth` app's permission to use an iAP2 link, not as the switch that creates one. Flipping it on a unit whose `btstack` has no iAP2 RFCOMM server enables nothing; MU1329's `btstack` is such a unit (P-321). Check MU0678's `btstack` for an iAP SDP record before testing the flag. | [bluetooth-iap2](bluetooth-iap2.md) §2.1, §9 |
| 2 | What endpoint path does `IapDeviceServices` return to `CIapBTChannel`? | On MH2p the Bluetooth endpoint is **`/dev/iapDevice-<16 hex digits of the BT address>`**, a resource manager created by `btstack` after the iAP2 detect bytes `FF 55 02 00 EE 10`, reported with `updIapDevicePath(addr, path)` and published by `bluetooth` as `rfCommDeviceName` (P-303, P-305). MH2p's `iap` app is the descendant of MU0678's `CIapBTChannel`, but nothing starts it (P-325). | MU0678's value is probably a `btstack` node of the same kind. Search MU0678 `btstack` for `/dev/iapDevice-`, `updIapDevicePath` and `IapServices`. | [bluetooth-iap2](bluetooth-iap2.md) §2.2-2.3, §7 |
| 3 | Is that endpoint DIO's `/dev/ipod0`? | **No** on MH2p. The Bluetooth node is owned by `btstack`, and the wired node is `/dev/otg-cinemo`; each transport has its own node (P-303, P-318). | Supports keeping E-023's separation. Nothing in a production design makes Bluetooth pretend to be the USB node. | [bluetooth-iap2](bluetooth-iap2.md) §9 |
| 4 | How does `MDNS_DIRECTLINK_IFACE=carplay0` reach `mdnsd`? | MH2p: as a shell prefix on the `mdnsd` command in `start_mdnsd()` (`/etc/boot/services.sh:364-368`), started once at boot. MU1329 does it differently: `dio_manager` spawns `mdnsd` per session with `dio_manager.json` `mdnsd.env` (P-111). | Two production designs differ, so MU0678 has its own producer. Search **all** IFS files for `MDNS_DIRECTLINK` (on MH2p, `dumpifs -x` skipped the very script that has it), and DIO's `startMdnsdProcess` (E-043). For wireless the variable does not matter: MH2p serves Wi-Fi as an ordinary multicast interface (P-219). | [configuration-and-boot](configuration-and-boot.md) §4.3 |
| 5 | Where is the AirPlay object's `interfaceName` populated? | By `dio_manager`, right after `AirPlayReceiverServerCreate`, via `CFObjectSetPropertyCString(server, …, AirPlayReceiverServerSetProperty, …, "interfaceName", name)`. MH2p value: `carplay0` for USB; for Wi-Fi the interface that owns `wifi.ipaddr` 10.174.189.1 (via `getifaddrs`), fallback `bcm0` if `/tmp/brcm_wifi` exists, else `uap0` (P-409, P-420). MU1329 sets the constant `carplay0` (P-423). | Look for the same `CFObjectSetPropertyCString` + `interfaceName` pattern in MU0678 `dio_manager`; E-026's object+0x6c field is that property's storage. | [airplay-dio](airplay-dio.md) §2.5, §3.1 |
| 6 | Callers and arguments of the packet/multicast interface helpers? | Not traced. MH2p still has `SocketSetMulticastInterface`, `SocketSetPacketReceiveInterface` and the stub-sized `SocketSetBoundInterface`. The binding that matters for Bonjour is `if_nametoindex(interfaceName)` → `DNSServiceRegister`, with a global `mdnsd` (P-404, P-111). | The wireless path does not depend on these helpers for discovery. Lower priority than it looked. | [airplay-dio](airplay-dio.md) §5 |
| 7 | How are Bluetooth/iAP2 and Wi-Fi/AirPlay tied to the same phone? | By the **phone's Bluetooth MAC**. SI starts `dio_manager` with `localBtMacAddress` and `wifiInfo`. `dio_manager` connects only to the `_carplay-ctrl._tcp` controller whose device ID equals that MAC (`CarPlayControllerGetBluetoothMacAddress`, 5 retries). SI also merges USB/BT identities via iAP2 `DeviceTransportIdentifierNotification`. The head unit's own BT MAC is in the Wi-Fi Apple IE and in the iAP2 Bluetooth transport component (P-211, P-307, P-311, P-313, P-314, P-422). | This is the correlation key TRACE-006 is looking for. | [airplay-dio](airplay-dio.md) §3.3; [bluetooth-iap2](bluetooth-iap2.md) §6 |
| 8 | Can AirPlay/mDNS work on `uap0` unmodified? | MH2p uses one code path for `carplay0` and the AP; the interface is only the `interfaceName` value, and `dio_manager` even names `uap0` as its fallback (P-420). What the AP needs is: `mdnsd` covering it, pf holes for 5353 and the CarPlay ports, a 5 GHz WPA2 AP, and the Apple IE (P-113, P-205..P-211, P-219, P-220). | Interface binding: probably yes. The session: **no**. MHI2-era receivers lack HomeKit pairing, the `_carplay-ctrl` connect trigger and iAP2-over-AirPlay; MU1329's 210.81 has none of the three (P-403, P-407, P-411). Configuration: MU1329's pf blocks 5353/7000 on `uap0` and runs `mdnsd` per session. | [wifi](wifi.md) §8; [airplay-dio](airplay-dio.md) §6 |
| 9 | Which production branch activates the wireless machinery into DIO? | There is no single flag. The chain is: `bluetooth.enableIap=true`; SI `bluetooth.enableDeviceDetection=true`; the phone's iAP2 `WirelessCarPlayUpdate`; coding (`carplayWireless` in SI, persistence key 8877 byte 0x17 bit 3 in `dio_manager`); a 5 GHz AP; then SI's `_carplay-ctrl._tcp` browse on the AP starts `dio_manager` with connection type Wi-Fi. DIO is never handed the Bluetooth link (P-102, P-105, P-107, P-309, P-310, P-313, P-425). | The activation lives in the orchestrator (SI) and a separate bootstrap client, not in DIO. MU0678 and MU1329 have no SI `wireless` block and no bootstrap process. | [configuration-and-boot](configuration-and-boot.md) §5-6 |
| 10 | What transport object is selected after the Bluetooth endpoint's `open64()`? | A **plain file descriptor on a byte-stream resource manager**. `iap2connectionmanager` `CIAP2TransportBT` (`open64(path, O_RDWR)` + `select`/`read`/`write`) runs the iAP2 link layer itself; so does the Cinemo plugin `CEsoIAPOverBTTransport` used by `media` (P-307, P-317). | Expect the same on MU0678: the userland client owns the link layer. | [bluetooth-iap2](bluetooth-iap2.md) §3, §4 |
| 11 | Same as 2: is it the mounted `ipod-drvr-iap2.so` service? | On MH2p the Bluetooth node belongs to `btstack`, not to an iAP2 driver (P-303). | See 2 and 12. | [bluetooth-iap2](bluetooth-iap2.md) §9 |
| 12 | Does the Bluetooth endpoint speak the 20-byte `iap2_connect()`/`iap2_msg()` ABI (0x9999)? | **No** on MH2p. It carries raw iAP2 link bytes (`io_read` blocks for RFCOMM data, `io_write` → `RF_SendData`); no control-message ABI is in `btstack`'s iAP resource manager (P-303). | TRACE-012's ABI belongs to a driver that contains the iAP2 link layer (`ipod-drvr-iap2.so`). The Bluetooth node is one layer lower. Pointing `libiap2client` at a Bluetooth node would not work without such a driver on top. | [bluetooth-iap2](bluetooth-iap2.md) §9 |
| 13 | How could Bluetooth iAP2 reach DIO when DIO is configured for `/dev/ipod0`? | MH2p never does this. The bootstrap iAP2 ends in `iap2connectionmanager`/SI, and DIO gets its iAP2 over AirPlay (`iAPSendMessage`) once the Wi-Fi session runs (P-306..P-316). | Suggests a **separate** small Bluetooth iAP2 client for the bootstrap (credentials, identification) instead of redirecting DIO's `/dev/ipod0` client to Bluetooth. | [bluetooth-iap2](bluetooth-iap2.md) §6, §10 |

## Roadmap items

| Roadmap item | MH2p answer |
| --- | --- |
| 2. Resolve `enableIap` parsing and control flow | Q1 above (P-300). |
| 2. Trace Bluetooth iAP → iAP2 transport creation | `btstack` RFCOMM → `/dev/iapDevice-*` → ASI path → userland iAP2 client `open64` (P-301..P-307). |
| 2. Production HCI transport | MH2p: UART `/dev/serbt1` at 3 Mbaud, BCM4349B1 patchram (P-117; [configuration-and-boot](configuration-and-boot.md) §4). MHI2's Marvell part uses SDIO; not comparable. |
| 2. iAP2 callbacks/events into DIO | Not over Bluetooth. DIO receives iAP2 over AirPlay after the session starts (P-315, P-316). |
| 3. Is the DIO iAP2 service transport-neutral? | MH2p DIO has **two** iAP2 clients: Cinemo for USB and ESO `CIAP2ServiceEso` for AirPlay-tunnelled iAP2. Cinemo itself is transport-pluggable through `CINEMO_OPTION_IAP_TRANSPORT_LIBRARIES` (P-315, P-319). |
| 3. Adaptation point for wireless iAP2 | `AirPlayReceiverSessionSendiAPMessage` / the `iAPSendMessage` command (P-315, P-411). |
| 4. Trace final interface selection into socket setup | `interfaceName` → `if_nametoindex` → `DNSServiceRegister`; `dio_manager` chooses the name (P-404, P-420). |
| 4. Correlate mDNS discovery with DIO session events | SI's `_carplay-ctrl._tcp` browse on the AP starts DIO; DIO's `CarPlayControlClient` matches by BT MAC (P-313, P-314, P-422). |
| 5. Session correlation | The BT MAC is the key (Q7). |
| 6. Media | Wireless audio adds Opus 16/24/48 kHz mono (P-413); `dio_manager.json` has separate `*AudioWireless` fragment sizes (P-414). Screen stream 110 is the same as wired (P-410). |
| 7. Implementation | See [requirements.md](requirements.md). |

## Traces

| Trace | What MH2p adds |
| --- | --- |
| TRACE-001 Bluetooth → iAP2 | Complete statically on MH2p: `btstack` `FUN_0015d270` → `FUN_0015cd1c` (`/dev/iapDevice-*`) → `updIapDevicePath` → `bluetooth` → `BluetoothMediaBridge` → SI → `iap2connectionmanager` `FUN_0004a21c` `open64` ([bluetooth-iap2](bluetooth-iap2.md) §9). |
| TRACE-002 DIO → iAP2 transport | DIO's wireless iAP2 is not a transport of the USB client; it is a second client over AirPlay (P-315). |
| TRACE-003 `MDNS_DIRECTLINK_IFACE` | MH2p producer: boot script shell prefix (P-111). Not needed for Wi-Fi (P-219). |
| TRACE-004 DIO → AirPlay | MH2p `FUN_0008f8f8`: Create → `interfaceName` → properties → `CarPlayControlClientCreateWithServer`/`Start` (P-420). |
| TRACE-005 AirPlay → socket/interface | Bonjour by interface index; the socket helpers' callers are still untraced (P-404). |
| TRACE-006 session correlation | BT MAC (Q7). |
| TRACE-007 / TRACE-008 multi-transport, control plane | MH2p shows the production form: component ids 97 (USB device), 98 (CarPlay over Wi-Fi), 99 (Bluetooth); Wi-Fi config answered with the live AP credentials (P-307, P-309). |
| TRACE-009 iAP2-NCM | Wired only on MH2p too (`usblauncher_carplay_descriptor.lua`, P-123). |
| TRACE-011 / TRACE-012 | The 0x9999 client ABI is not used for Bluetooth on MH2p (Q12). |

## Evidence-register entries this touches

| MHI2 entry | MH2p relation |
| --- | --- |
| E-006, E-024 (`enableIap=false`) | MH2p has `true`, with its consumer identified (Q1). |
| E-018, E-019, E-027 (screen setter and bind stubs) | The same stubs are in MU1329's 210.81 (P-402); 320.17.1 removed them. |
| E-026 (`interfaceName` → `if_nametoindex`) | Same mechanism in 210.81 and 320.17.1 (P-404); the producer is identified (Q5). |
| E-031, E-038, E-043 (mDNS environment) | Q4. |
| E-036, E-052 (`iap2.cfg`) | MH2p has no `iap2.cfg`; Bluetooth link parameters are in `iap2connectionmanager.json` (`bt` block: 5/2048/3000/700/30/3 vs USB 5/4096/2000/100/30/3, P-308). |
| E-041 (accessory Wi-Fi configuration response) | MH2p's production use: SSID, passphrase, security, channel of the 5 GHz internal AP (P-215, P-309). |
| E-049..E-055 (`iap2_connect` ABI) | Not part of MH2p's wireless path (Q12, Q13). |
