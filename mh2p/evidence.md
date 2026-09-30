# MH2p Evidence Register

Central index of the `P-` evidence entries. Each subsystem document keeps its own table next to the
analysis; this file collects them. **If an entry here and its subsystem document disagree, the subsystem
document wins** (see [source-of-truth](../docs/source-of-truth.md)): it is where corrections are made.

All entries are **static** evidence from the MH2p baseline in [firmware.md](firmware.md). No MH2p unit
was observed at runtime. Status vocabulary:

- **Proven** — directly shown by MH2p configuration, strings, symbols or disassembly.
- **Partial** — the components or boundary are shown, part of the path is not.
- **Inferred** — an interpretation that is not directly shown.

An MH2p entry is **cross-platform evidence**. It never proves MHI2 (MU0678) or MHI2Q (MU1329) behaviour;
entries about MU1329 are marked as such in their finding and were checked against the MU1329 firmware.

Corrections made while consolidating (2026-09-28):

- P-217 originally said `dio_manager` reads `WIFI_IFNAME` first. The decompile (P-420) shows it does not:
  the Wi-Fi interface is the one that holds `WIFI_IPADDR`, with a `bcm0`/`uap0` fallback.
- The configuration agent's Q2/Q3 and Q7 rows were string-level inferences; P-303/P-307 and P-314/P-422
  prove them by disassembly.
- MHI2Q MU1329 is **Qualcomm APQ8064**, QNX 6.5.0 SP1 (P-470), not NVIDIA Tegra 3.

## Configuration and boot (P-100..P-199) — [configuration-and-boot.md](configuration-and-boot.md)

| ID | Finding | Class | Status | Source |
| --- | --- | --- | --- | --- |
| P-100 | SI `children.carplay.wireless` = `mDNSDiscovery{regType "_carplay-ctrl._tcp", wifiInterfaceName "bcm0"}`, `enable5GHzCountryAdaption true`; the only wireless block | Config | Proven | `efs-system/etc/eso/production/smartphone_integrator.json` |
| P-101 | `dio_manager.json` has `wifi{force false, ifname bcm0, ipaddr 10.174.189.1}`, `bt{connectionTimeoutMs 2000, fakeMacAddr}`, `*AudioWireless` frag sizes, and an OPUS-over-WiFi comment; `iap2.device` stays `iap://ffs:///dev/otg-cinemo`; no mdnsd block | Config | Proven | `efs-system/etc/eso/production/dio_manager.json` |
| P-102 | SI `bluetooth.enableDeviceDetection true`, `iap2ResetTimeout 7000`; thread `dnsServiceDiscoveryService` | Config | Proven | `smartphone_integrator.json` |
| P-103 | SI child `iap2connectionmanager`; `iap2connectionmanager.json` defines separate `usb` and `bt` iAP2 link parameters; the binary has BT/USB/CoW transports and Wi-Fi info-sharing jobs; absent on MU1329 | Config + Static | Proven | `iap2connectionmanager.json`, `eso/bin/apps/iap2connectionmanager` |
| P-104 | SI's DNS-SD browse callback drops services not on the configured Wi-Fi interface (`if_indextoname` compare) | Disassembly | Proven | `smartphone_integrator` `dnsServiceCarPlayBrowseReply` @ Ghidra `0xddfb4` |
| P-105 | SI gates wireless CarPlay on a 5 GHz AP ("5GHz access point is not available to activate Wireless CarPlay"), and on coding `carplayWireless` | Static | Partial | SI strings; ref from `FUN_000a7cb8` |
| P-106 | `btstack` creates `/dev/iapDevice-<addr>` after an iAP2 handshake and reports it via `updIapDevicePath` to `bluetooth` | Static | Partial | `eso/bin/apps/btstack`, `bluetooth` strings |
| P-107 | `connectivity.json`: `bluetooth.enableIap true` (MU1329 `false`); `connectionmanager` preCondition `/tmp/uap0`; `btstack` preCondition `/dev/serbt1`; `wifidriver qwdi` | Config | Proven | `efs-system/etc/eso/production/connectivity.json`; MU1329 `connectivity.json:216` |
| P-108 | `connectionmanager` runs `hostapd-2.5-wapi` per AP from templates `ap0.config` (5 GHz, 11ac) and `ap1.config` (2.4 GHz), WPA2-CCMP; owns dnsmasq and pf switching | Static + Config | Partial | `connectionmanager` strings; `app/eso/telephone/ap{0,1}.config`, `restart_dnsmasq.sh`, `switchpf.sh` |
| P-109 | `start_wifi_driver`: BCM4359 qwdi; `bcm0` 10.174.189.1 AP (5 GHz, `wl ap 1`) → `/tmp/uap0`; `bcm1` 10.173.189.1 (2.4 GHz) → `/tmp/uap1`; `bcm2` STA; `/tmp/brcm_wifi` = bcm0 MAC | Static (script) | Proven | `stage2_ifs4/etc/boot/drivers.sh:94-165` (sliced) |
| P-110 | `dio_manager` selects the AirPlay interface: `carplay0` (wired) or, for WiFi transport or `wifi.force`, the interface owning `wifi.ipaddr`, else `bcm0`/`uap0` by `/tmp/brcm_wifi`; sets it on the AirPlay server | Disassembly | Proven | `dio_manager` Ghidra `FUN_0008f8f8`, `FUN_000a8038`, `CDioManagerComp::toString` |
| P-111 | mdnsd is boot-started in normal mode with `MDNS_DIRECTLINK_IFACE=carplay0 mDNSPlatformDefaultProbeCountForTypeUnique=0 ... mdnsd -U mdnsd`; not per session (SI cleanupScript `""`; no mdnsd code in MH2p `dio_manager`) | Static (script) | Proven | `stage2_ifs4/etc/boot/services.sh:364-368`, `startup.normal.sh` |
| P-112 | SI and `dio_manager` both link `libdns_sd.so.1`; MH2p mdnsd supports `MDNS_EXCLUSIVE_IFACE` (unset) as well as `MDNS_DIRECTLINK_IFACE` | Static | Proven | ELF NEEDED; `armle/usr/sbin/mdnsd` strings |
| P-113 | pf passes wireless CarPlay TCP/UDP ports and mDNS multicast on both `bcm0` and `bcm1`; loaded after `/tmp/uap0` and `/tmp/uap1`; same rules in `pf.ecm0.conf` and `pf.mlan0.conf` | Config | Proven | `stage2_ifs4/etc/pf.conf:97-211` (sliced), `services.sh:58-92` |
| P-114 | MU1329 pf passes no 5353 and no CarPlay ports on `wlan_if`; the CarPlay ports are only on `carplay0` | Config | Proven | MU1329 `system/etc/pf.conf:90-140,195-203` |
| P-115 | MU1329 `dio_manager.json` `mdnsd.env.directLink "MDNS_DIRECTLINK_IFACE=carplay0"`; MU1329 `dio_manager` spawns and stops mdnsd; `carplay_cleanup.sh` slays mdnsd | Config + Static | Proven | MU1329 `dio_manager.json:15-34`, `dio_manager` strings, `scripts/carplay_cleanup.sh` |
| P-116 | `dnsmasq.conf` serves 10.173.189.128-247 and 10.174.189.128-247; `hosts` names `hotspot5.mib2p` 10.174.189.1 | Config | Proven | `stage2_ifs4/etc/dnsmasq.conf`, `etc/hosts` |
| P-117 | `start_bluetooth_driver` waits for Wi-Fi (`/tmp/mvloaded`), patchram BCM4349B1, `/dev/serbt1` | Static (script) | Proven | `drivers.sh:173-193` |
| P-118 | `startup.production.sh` (factory mode) does not start mdnsd | Static | Proven | `stage2_ifs4/etc/boot/startup.production.sh` |
| P-119 | Presets (`dio_manager`, `smartphone_integration`, `bluetooth`, `networking`) are log-level only | Config | Proven | `efs-system/etc/eso/presets/*.json` |
| P-120 | `framework.json`: SI and connectivity_launcher have exec; dio_manager, iap2connectionmanager and connectionmanager are `exec:null` (spawned by launchers); dsistartup runs connectivity_launcher then SI | Config | Proven | `stage2_ifs3/etc/eso/production/framework.json`, `dsistartup.json` |
| P-121 | `extdevconfig.json` holds MFi auth (`/dev/i2cmfi:34`), identification (ALPINE, `MIB2+`, AUDI MMI) and EAP; moved out of `dio_manager.json` compared with MU1329 | Config | Proven | `efs-system/etc/eso/production/extdevconfig.json` |
| P-122 | The dedicated CarPlay io-pkt (`/cplay`, `io-pkt1`) is supported by scripts but disabled (`useForCarPlay 0`); only one io-pkt is started | Config + Static | Proven | SI `usb.dedicatedIoPkt`, `startncm.sh`, `main_stage2.1.sh:61` |
| P-123 | Wired path: usblauncher_otg → `io-otg-cinemo` (`/dev/otg-cinemo`) + `startncm.sh` → `carplay0`; descriptor VID/PID `0x1C98/0x2002`, "iAP2 NCM Accessory"; no wireless descriptor | Config + Static | Proven | `usblauncher_otg.lua:85-250`, `usblauncher_carplay_descriptor.lua`, `start_io-otg-cinemo.sh`, `startncm.sh` |

## Wi-Fi access point (P-200..P-299) — [wifi.md](wifi.md)

| ID | Finding | Class | Status | Source |
| --- | --- | --- | --- | --- |
| P-200 | `start_wifi_driver` mounts `devnp-qwdi-2.5_bcm4359-wapi.so` with `fw_bcmdhd_4359.bin`, `nvram_4359_c_samples.txt` (`nvram_4359.txt` if REVISION<200), `4359b1.clm_blob` | Config | Proven | `/etc/boot/drivers.sh:94-165` (stage2_ifs4.img) |
| P-201 | Interfaces `bcm0` 5 GHz AP 10.174.189.1/24, `bcm1` 2.4 GHz AP 10.173.189.1/24, `bcm2` station, `bcm_p2pdev0`; `wl ap 1`, `wl frameburst 1` | Config | Proven | `drivers.sh:131-160`, `app/eso/bin/reset_wlan.sh`, `/etc/dhcp/dhcp-check` |
| P-202 | Marker files `/tmp/uap0`, `/tmp/uap1`, `/tmp/mvloaded`, `/tmp/brcm_wifi` keep Marvell-era names | Config | Proven | `drivers.sh`, `services.sh:83,105-106` |
| P-203 | Marvell fallback AP path (`uaputl` uap0 2.4 GHz ch 6, uap1 5 GHz ch 36) present but `uaputl` not shipped | Config/Static | Proven | `app/eso/telephone/start-ap.sh`, `start-ap5g.sh`, `switchpf.sh`; `app_list.txt` |
| P-204 | `connectionmanager` generates hostapd configs from `/eso/telephone/ap<N>.config` into `/tmp/ap.config.bcm<N>`, runs `hostapd-2.5-wapi` per AP | Static/Disassembly | Proven | `connectionmanager` strings 0x106f64-0x107400, code 0xa91f2, 0xa84d8 |
| P-205 | 5 GHz template: `hw_mode=a`, 11n HT40+, 11ac with 80 MHz off, WMM, `max_num_sta=8`; no country/vendor_elements | Config | Proven | `app/eso/telephone/ap0.config` |
| P-206 | Security WPA2-PSK CCMP (`wpa=2 ... rsn_pairwise=CCMP`), WAPI alternative | Static | Proven | `connectionmanager` 0x107078-0x107177 |
| P-207 | 5 GHz channel per country from `mcc_countrycode.xml`: only 36 or 149 (non-DFS); disabled where `active5Ghz="false"` | Config/Static | Proven | `<MH2p SWDL>/Data/WLAN.Config_*/0/connectivity/mcc_countrycode.xml`; `connectionmanager` 0xfebc0, 0xfd588 |
| P-208 | Country via `wl country`; channel moves via `wl -i bcm0 csa b 1 %d`; `wl rsdb_mode` | Disassembly | Proven | `connectionmanager` 0x7aff2, 0x79c96, 0x7ad98 |
| P-209 | Default SSID prefixes per brand (`Audi_MMI_`, ...), ` 5GHz` suffix, random lowercase passphrase; sample `Audi_MMI_**** 5GHz` / `****-****-****` (masked) | Static | Partial | `connectionmanager` 0xfd8b8-0xfd920, 0xfdc74, 0xfdd30, 0x73e12; `smartphone_integrator` 0xecf98 |
| P-210 | Apple vendor IE set with `wl -i <ifc> add_ie 7 <len> 00:a0:40 <hex>` on `bcm0`/`bcm1`, refreshed on carrier-up and BT-address update | Disassembly | Proven | `connectionmanager` 0x7b370-0x7b9a0, callers 0x7bbbe, 0x7c456, 0x7c470, 0x7c770 |
| P-211 | Apple IE payload: OUI type 0; TLV 0 flags (2 B, bit 0x01 = 5 GHz, 0x20/0x10 state bits); TLV 1 `MIB2P`; TLV 3 `A8`; TLV 2 `Audi AG`; TLV 6 BT MAC | Disassembly | Partial (flag semantics open) | `connectionmanager` 0x7b4da-0x7b8fc |
| P-212 | Interworking IE 107 (venue 0x0A/0x01) and link tuning `srl/lrl 15`, `ampdu_rts 0`, `scb_activity_time 1` | Disassembly | Partial (IE decoding inferred) | `connectionmanager` 0x79800-0x79950, 0x7bb60-0x7bd90 |
| P-213 | Wireless CarPlay uses `bcm0`: `wifiInterfaceName "bcm0"`, `wifi.ifname "bcm0"`, `ipaddr "10.174.189.1"`; Android Auto internal AP `bcm0`, customer AP `bcm1` | Config | Proven | `efs-system/etc/eso/production/smartphone_integrator.json:203-210`, `dio_manager.json:364-368`, `gal.json:576-582` |
| P-214 | `smartphone_integrator` refuses wireless CarPlay when the 5 GHz AP is unavailable (`enable5GHzCountryAdaption`) | Static/Config | Proven (strings + config) | `smartphone_integrator` 0xee41c, 0xec874, 0xec8dc; `smartphone_integrator.json:209` |
| P-215 | Credential path: `connectionmanager` WlanService (`updateinternalApDetails`) -> `smartphone_integrator` WiFiAPInfoProvider -> `iap2connectionmanager` (`publishWiFiAccessPointInformation` / `onJob_iap2AccessoryWiFiConfigurationInformation`) -> `libesoiap2.so` WifiShare -> 0x5703 | Static | Partial (edges from strings/symbols; no call trace) | `connectionmanager` 0x110464; `smartphone_integrator` 0xe3710-0xe3dc8, 0xf05b0, 0xf06ec, 0xf0fe8, 0xf1058; `iap2connectionmanager` 0x42658, 0x732c; `libesoiap2.so` symbols; `libasimmxnetworkingproxy.so`, `libasimmxiap2connectionproxy.so` IDL names |
| P-216 | `dio_manager` does not use the WlanService ASI | Static | Proven (absence in strings) | grep of all `app/` binaries for `networking.WlanService` |
| P-217 | `dio_manager` Wi-Fi ifname: the interface holding `WIFI_IPADDR`, else `bcm0` if `/tmp/brcm_wifi` exists, else `uap0`. Corrected: `WIFI_IFNAME` is not read here (decompile, P-420 in airplay-dio.md) | Disassembly | Proven (corrected) | `dio_manager` 0x7f940-0x7fd44, strings 0xc6100-0xc6160, 0xb95e4-0xb95fc |
| P-218 | dnsmasq `-i bcm0 -i bcm1`; bcm0 range 10.174.189.128-.247, 3 d; router option suppressed without uplink | Config | Proven | `/etc/dnsmasq.conf`, `services.sh:100-109`, `app/eso/telephone/modWLANserver.sh` |
| P-219 | `mdnsd` started at boot with `MDNS_DIRECTLINK_IFACE=carplay0 mDNSPlatformDefaultProbeCountForTypeUnique=0`, no interface restriction | Config | Proven | `services.sh:364-368`, `startup.normal.sh:41` |
| P-220 | pf on both APs: CarPlay TCP 6030 + `5000:5001,5010,6000:6001,6100,6200,7000:7001,7100`, UDP set incl. 5353, mDNS v4/v6, ICMPv6; per-AP ALTQ; inter-AP traffic blocked | Config | Proven | `/etc/pf.conf:5-6,50-51,98-212` |
| P-221 | MU1329 `connectionmanager` has no Apple-IE / custom-IE code; MU1329 `uaputl` supports `sys_cfg_custom_ie`, `sys_cfg_channel_ext` (5 GHz band), `cfg_80211d`, `sys_cfg_wmm`, but no `vhtcfg` | Static | Proven | `<MU1329>/app/eso/bin/apps/connectionmanager`, `app/armle/sbin/uaputl` strings |
| P-222 | MU1329 `pf.conf` on `uap0` has no pass for 5353 or AirPlay ports | Config | Proven | `<MU1329>/system/etc/pf.conf:97-137` |

## Bluetooth bootstrap and iAP2 (P-300..P-399) — [bluetooth-iap2.md](bluetooth-iap2.md)

| ID | Finding | Class | Status | Source |
| --- | --- | --- | --- | --- |
| P-300 | `bluetooth` reads `bluetooth.enableIap` and `global.wirelessCarPlay` into `CBT_CONFIG` (`iapEnabled`, `wirelessCarplayEnabled`); `CBluetoothIapManager` enforces iAP2 allowed/connect and wireless-CarPlay reconnect | Static / Disassembly | Partial (field → branch not followed) | `app/eso/bin/apps/bluetooth` `FUN_0006c094`, `FUN_0006c318`, `FUN_00061f5c`, `FUN_0006fa44`, `FUN_0006f4a0`; `connectivity.json` |
| P-301 | `btstack` registers an iAP SDP record: ServiceClassID `00000000-deca-fade-deca-deafdecacaff`, L2CAP+RFCOMM | Static / Data / Disassembly | Proven | `btstack` file `0x1f9230..0x1f9253`; `FUN_0015884c` |
| P-302 | `btstack` SDP/EIR-checks the phone for iAP2 (`…cacafe`) and Wireless CarPlay `2D8D2466-E14D-451C-88BC-7301ABEA291A` | Static / Data | Proven | `btstack` `0x1f90bc`, `0x1f913f` (reversed), `0x1a728c`, `0x1a72d8`; `FUN_00071c34` |
| P-303 | After `FF 55 02 00 EE 10` is seen on RFCOMM, `btstack` creates resmgr `/dev/iapDevice-%016llX` (byte stream) and calls `updIapDevicePath(addr, path)` | Disassembly | Proven | `btstack` `FUN_0015d270`, `FUN_0015cd1c`, `FUN_0015d74c`; constant `0x1e0940` |
| P-304 | `WirelessCarPlayServices::connectService` is `NOT SUPPORTED`: no BT data channel beyond iAP2 | Static | Proven (string) | `btstack` `0x1a542c..0x1a548c` |
| P-305 | `bluetooth` stores the path per device (iAP2 bit `0x40000000`) and serves `BluetoothMediaBridge` whose device list carries `rfCommDeviceName`; SI and `iap` are clients | Static / Disassembly | Proven (server/client), Partial (list content) | `bluetooth` `FUN_00046028`, `FUN_00070e10`, `BluetoothMediaBridgeS.hxx`; SI `FUN_000bed98` |
| P-306 | `iap2connectionmanager` is an SI child on `libesoiap2` (no Cinemo), serving `asi.media.iap2connection.IAP2Connection` | Static / Config | Proven | ELF NEEDED; `smartphone_integrator.json`; strings `0x43518` |
| P-307 | `CIAP2TransportBT` does `open64(path, O_RDWR)`. Identification: BT component id 99 `ESO-BT-Transport` + MAC, USB device id 97 `ESO-USB-Device-Transport`, WirelessCarPlay id 98 `ESO-CoW-Transport` (flag) | Disassembly | Proven | `iap2connectionmanager` `FUN_0004a21c`, `FUN_00040b58` |
| P-308 | BT iAP2 link params 5/2048/3000/700/30/3 vs USB 5/4096/2000/100/30/3 | Config | Proven | `iap2connectionmanager.json` |
| P-309 | SI connects with `enableWirelessCarPlayFeatures` and sends `iap2AccessoryWiFiConfigurationInformation` {SSID, passphrase, channel, security 0/1/2} | Disassembly | Proven (send), Partial (connect parameters) | SI `FUN_000b303c`, `FUN_000b338c`, `FUN_000b0be8` |
| P-310 | SI enables wireless CarPlay per device only if the phone is BT-connected and the 5 GHz AP is available | Static | Proven (strings, function) | SI `FUN_000a7cb8` |
| P-311 | SI merges USB/BT identities of a phone via `DeviceTransportIdentifierNotification`; the device model holds `rfcommDeviceName` | Static | Partial | SI strings `0xee2a0..0xee35c`, `FUN_000a4f20`, `FUN_000a54bc` |
| P-312 | `dio_manager` Cinemo (wired) identifies USB, BT (local MAC) and Wi-Fi components and handles OOB BT pairing, WirelessCarPlayUpdate and DeviceTransportIdentifierNotification, relaying them to SI | Disassembly / Static | Proven (components, relay strings) | `dio_manager` `FUN_000ae838`, `FUN_000aeb14`, `FUN_000aee08`, `FUN_000ade7a`, `FUN_000ad90e`; `CSIService` strings |
| P-313 | SI browses `_carplay-ctrl._tcp` and starts `dio_manager` for a Wi-Fi device with `wifiInfo [ipAddress, macAddress, port, transportDeviceName]` | Static / Config | Partial | SI `CDNSServiceDiscoveryService`, `FUN_000db7ec`; `dio_manager` `FUN_00072b60`; P-100 |
| P-314 | `dio_manager` creates the AirPlay server and a `CarPlayControlClient`, and connects only to the controller whose BT MAC matches the target | Disassembly | Proven | `dio_manager` `FUN_0008f8f8`, `s_carPlayControlClientEventCallback` (0x8f0d8), `FUN_0008e220` (`STALeft`) |
| P-315 | `dio_manager` sends iAP2 via `AirPlayReceiverSessionSendiAPMessage`; `CIAP2TransportHandler` feeds received bytes to the `libesoiap2` link of `CIAP2ServiceEso` | Disassembly | Proven | `dio_manager` `FUN_0008769a`, `FUN_000b7b9a`, `FUN_000b70a0` |
| P-316 | iAP2 over AirPlay is the `iAPSendMessage` command, not a stream type | Static | Proven (string/API) | `eso/lib/libairplay.so` `0xc3568`, `0xc3b98` |
| P-317 | `libEsoIAPTransport.so` is a Cinemo iAP transport plugin (`CreateEsoIAPOverBTTransport`, `esoiapoverbt_*`) for `bt://<path>` with A2DP audio from ASI | Static (exports, strings) | Proven | `app/armle/usr/lib/cinemo/libEsoIAPTransport.so` |
| P-318 | `media` loads it through `CINEMO_OPTION_IAP_TRANSPORT_LIBRARIES` and uses `iap://bt://…`; `dio_manager` does not (USB `iap://ffs:///dev/otg-cinemo` only) | Static / Config | Proven (imports, strings, config); option value Inferred | `media`, `media.json` (`pluginNameIapTransport`, `bluetooth.iapSupport`, `EnableOOBBTPairing`, `EnableWiFiInformationSharing`), `dio_manager.json` |
| P-319 | Cinemo's `NmeTransportFactory` loads iAP transports from `IAP_TRANSPORT_LIBRARIES` and validates the `ctli_context` function table; `libNmeTransport` exports `CreateNmeIAPUSBTransport` and `CreateNmeIAPQRMTransport` | Static | Proven | `app/armle/usr/lib/libNmeBaseClasses.so`, `cinemo/libNmeTransport.so` |
| P-320 | MU1329 Cinemo has the same `IAP_TRANSPORT_LIBRARIES` mechanism with an older `ctli_context` ABI and only the `usb`/`ffs` transports | Static | Proven (strings/symbols) | `<MU1329>/app/armle/usr/lib/libNmeBaseClasses.so`, `libNmeSDK.so`, `cinemo/libNmeTransport.so` |
| P-321 | MU1329 `btstack` has no iAP SDP record, iAP2/wireless UUIDs, detect bytes or `/dev/iapDevice`. MU1329 `bluetooth` has `bluetooth.enableIap` and `SERVICETYPE_IAP2`; `connectivity.json` sets `false` | Static / Config | Proven (absence by byte search) | MU1329 `app/eso/bin/apps/{btstack,bluetooth}`, `system/etc/eso/production/connectivity.json:216` |
| P-322 | MH2p Cinemo `libNmeVfs` has Wi-Fi/WirelessCarPlay transport identification, OOB BT pairing and Wi-Fi info-sharing responders; MU1329 has only the Bluetooth component | Static | Proven (strings) | `libNmeVfs.so` MH2p vs MU1329 |
| P-323 | MU1329 ships `libasimmxconnectivity_bluetooth_iapproxy.so`, but no MU1329 app references `IapDeviceServices`; no `iap`, no `iap2connectionmanager` | Static | Proven (grep) | MU1329 `app/eso/lib/factories`, `app/eso/bin/apps` |
| P-324 | No `CarPlayAvailability`/`CarPlayStartSession` names in any MH2p iAP2 stack; the flow is Wi-Fi config + `_carplay-ctrl` + CarPlayControl connect | Static | Proven (absence of names) / Inferred (flow) | string search of `libesoiap2`, `libNmeVfs`, `libNmeSDK`, `dio_manager`, SI, `iap2connectionmanager` |
| P-325 | MH2p `iap` app is an ESO iAP2-over-BT client (`eso-Bt-iAP`, Wi-Fi AP sharing) with no launcher entry | Static / Config | Proven (strings, launcher config) / Inferred (dormant) | `app/eso/bin/apps/iap`; `connectivity.json`; `framework.json` |
| P-326 | The `bluetooth` app serves `BluetoothSmartphoneIntegration` (modes incl. `CARPLAY_WIRELESS`, OOB pairing replies); `dio_manager` `CBluetoothController` uses it and falls back to `bt.fakeMacAddr` after 2000 ms | Static / Disassembly | Proven (strings, call sites) | `bluetooth` `FUN_0007cc90`; `dio_manager` `FUN_00093960`, `s_btInterfaceTimerHandler`; `dio_manager.json` `bt` |

## AirPlay receiver and DIO (P-400..P-469) — [airplay-dio.md](airplay-dio.md)

| ID | Finding | Class | Status | Source |
| --- | --- | --- | --- | --- |
| P-400 | MH2p `libairplay.so` is `AirPlay/320.17.1`, GCC 4.9.2, NEEDED incl. `libopus`, `libnvmedia`, `libnvparser`, `libcpp-ne.so.5`; MU1329 is 210.81 with OMX and `libecpp-ne.so.4` | Static | Proven | `app/eso/lib/libairplay.so` 0xc8094, `.comment`, `DT_NEEDED`; MU1329 `app/eso/lib/libairplay.so` 0x9df2c |
| P-401 | Export diff: 1,914 vs 1,468 defined globals; 320.17.1 adds CarPlayControl, HomeKit pairing, SendiAPMessage, Opus/NvMedia families; 210.81 alone has NTPClient, AirTunes, HID*, APSMFi*, screen setters | Static | Proven | pyelftools `.dynsym` diff of both files |
| P-402 | MU1329 `AirPlayReceiverSessionScreen_SetIFName/SetTransportType/SetClientIfMACAddr` are 4-byte stubs; absent in 320.17.1 | Static | Proven | MU1329 `libairplay.so` `.dynsym` `st_size` |
| P-403 | 320.17.1 implements HomeKit pair-setup/pair-verify (SRP-3072-SHA512, Ed25519, Curve25519, ChaCha20-Poly1305, HKDF-SHA512), keychain `/mnt/misc1/carplay/Library/KeyChains/carplay.keychain`; 210.81 has none | Static | Proven | `libairplay.so` strings 0xc3014, 0xc3074, 0xd9a10..0xda4b0, 0xd84e0; MU1329 string counts |
| P-404 | `_UpdateBonjourAirPlay` registers `_airplay._tcp` with TXT `deviceid, features, fv, flags, model, pi, srcvers=320.17.1`, flags 0x800, ifIndex `if_nametoindex(obj+0x1a1)`; 210.81 uses obj+0x16c, flags 0 | Disassembly | Proven | MH2p `FUN_0003e0dc`; MU1329 `FUN_00030fb8` |
| P-405 | `_raop/_hap/_mfi-config/_airport` strings only referenced by BonjourBrowser/AsyncConnection utility and test code | Disassembly | Partial | Ghidra xrefs, `libairplay.so` 0xe08e4, 0xe0a04, 0xe0a18, 0xe0530 |
| P-406 | CarPlayControlClient browses `_carplay-ctrl._tcp` `local.` via `BonjourBrowser` | Disassembly | Proven | `FUN_0004f9f4` |
| P-407 | Connect = `GET /ctrl-int/1/connect` with `Host` and `AirPlay-Receiver-Device-ID`, 2 s timeout, scoped address, `wifi` flag | Disassembly | Proven | `FUN_00050330`, strings 0xc8074, 0xc80a8 |
| P-408 | `CarPlayControllerGetBluetoothMacAddress` = `BonjourDevice_GetDeviceID`; `GetInterfaceName` = service `ifname` | Disassembly | Proven | 0x50bb0, 0x50ca4 |
| P-409 | `AirPlayReceiverServerSetProperty` keys `deviceID` (6 B), `interfaceName` (17 B), `playing`, `model`; same keys in 210.81 | Disassembly | Proven | MH2p 0x3e93c (CFSTRs 0xc1fc4, 0xc1fd8, 0xc1ea8, 0xc1ee4); MU1329 0x318d0 |
| P-410 | SessionSetup accepts stream types 100/101/102/110 only; per-stream keys via `DataStream-Salt%llu` HKDF | Disassembly | Proven | `AirPlayReceiverSessionSetup` 0x47170, `FUN_00044dc4` callers 0x47424/0x479d6/0x4827c |
| P-411 | iAP2 over Wi-Fi uses `iAPSendMessage` command (HU→phone `AirPlayReceiverSessionSendiAPMessage`; phone→HU in `dio_manager` `handleSessionControl`, forwarded to `this+0x3b0`); `disableBluetooth` releases the phone's BT link | Disassembly | Proven | `dio_manager` `FUN_0008f3bc` (CFSTRs 0xc3f24..0xc3fb0 file) |
| P-412 | 320.17.1 has no stream-130 handling; whether iOS still offers the command form to such receivers is not determinable statically | Inference | Inferred | P-410, P-411, xcertplay 03-wireless.md |
| P-413 | 320.17.1 audio formats include `OPUS/16000/1`, `OPUS/24000/1`, `OPUS/48000/1`, mono AAC-ELD; Opus encode+decode; 210.81 has ALAC/AAC/PCM only | Disassembly + Static | Proven | SessionSetup format table; MU1329 strings 0x9d904..0x9da74 |
| P-414 | `dio_manager.json` wireless audio: separate `*AudioWireless` frag sizes (same values), OPUS 48 kHz mono comment; latencies not transport-specific | Config | Proven | `efs-system/etc/eso/production/dio_manager.json` `audio` |
| P-415 | Timing is NTP-style in both (320: `_TimingNegotiate`, `timingPort`; 210: `NTPClock*`); no PTP strings | Static | Proven (absence of PTP names) | strings; `FUN_00041e30` |
| P-416 | 320.17.1 adds `keepAliveLowPower`, `keepAlivePort`, `KeepAliveWithBody` | Static | Proven | strings 0xd20dc, 0xd32c8 |
| P-417 | `/info`/setup key set of 320.17.1; new vs 210.81: `extendedFeatures`, `keepAliveLowPower`, `oemIcons`, `vehicleInformation`, `redundantAudio`, `usingScreen`, `transportType` | Static | Partial (from strings) | string block 0xc2000..0xc9000 |
| P-418 | 320.17.1 video via NvMedia + QNX Screen; 210.81 via Qualcomm OMX | Static | Proven | NEEDED, log categories |
| P-420 | `dio_manager` AirPlay thread: Create → interfaceName (`carplay0` unless Wi-Fi or `WIFI_FORCE`; then interface owning `WIFI_IPADDR`; fallback `bcm0` if `/tmp/brcm_wifi` else `uap0`) → model → delegate → MaxFps → CarPlayControlClient → `startServer` | Disassembly | Proven | `FUN_0008f8f8`, `FUN_000a8038`, `toString` 0x68e10 |
| P-421 | SI connection type enum 0 UNKNOWN / 1 USB / 2 WIFI; interface selection tests type == 2 (Wi-Fi); AirPlay NetTransportType 1 Enet, 2 WiFi, 8 USB, 0x10 Direct, 0x20 BTLE, 0x40 WFD | Disassembly | Proven | `FUN_00072974`, `FUN_00096ee8`, `FUN_00072df4`, `s_sessionCreatedCallback` 0x911b8 |
| P-422 | `dio_manager` connects only to a `_carplay-ctrl` controller on its own interface whose BT MAC matches the target (adopts first if none), 5 retries | Disassembly | Proven | `s_carPlayControlClientEventCallback` 0x8f0d8 |
| P-423 | MH2p deviceID from a stored MAC string with bit 0 flipped (logged as BT MAC); MU1329 deviceID = `uap0` MAC and `interfaceName` = constant `carplay0`, model `AirPlayGeneric1,1` | Disassembly | Partial (MH2p source field) / Proven (MU1329) | MH2p `FUN_0008c0b4`, `FUN_000a6d14`, `FUN_00070d24`; MU1329 `FUN_0015c204` |
| P-424 | MH2p `dio_manager` has Cinemo Wi-Fi component, OOB BT pairing, Wi-Fi info sharing, WirelessCarPlayUpdate and DeviceTransportIdentifierNotification handling; MU1329 `dio_manager` has none | Static | Proven (strings) | `dio_manager` strings 0xc46ac..0xd36e8; MU1329 string counts |
| P-425 | `DIOCodingProvider` derives `enCarPlayTechnology` from persistence key 8877: byte 8 bit 0 = USB, byte 0x17 bit 3 = Wi-Fi | Disassembly | Partial (key's source) | `FUN_000a6624`, `FUN_000711b8`, `FUN_000711fc`, data words 0xcc968..0xcc990 |
| P-426 | MH2p HMI has `WirelessCarplayHMIModelHandler`, `WlanWaker`, `BluetoothWaker`, `SERVICETYPE_CARPLAY_OVER_WIRELESS`, technology selection; MU1329 HMI has only `CONNECTIONTYPE_WLAN` | Static | Proven (string counts) | `stage2_ifs5/ifs/lsd.jxe`; `<MU1329>/ifs/lsd.jxe` |

## Portability to MHI2 / MHI2Q (P-470..P-499) — [portability.md](portability.md)

| ID | Finding | Class | Status | Source |
| --- | --- | --- | --- | --- |
| P-470 | MU1329 is Qualcomm APQ8064 (not Tegra 3), QNX 6.5.0 SP1; MH2p is Tegra K1, QNX 6.6.0 | Static | Proven | `firmware/MMX2QC/qcbl`, `*.mbn`, eifs/ifs2 `lib*_8064.so`, `libadreno_utils.so`; MU1329 `ifs/mifs/proc/boot/libc.so.3` build path; MH2p `stage2_ifs3/lib/libcapture-soc-t124.so` |
| P-471 | MH2p CarPlay binaries: soft-float EABI5, VFPv3-D32 + NEON, Thumb-2; MU1329 already ships VFPv3+NEON binaries (e.g. `libNmeSDK.so`) | Static | Proven | `readelf -h -A`; attribute scan of MU1329 extract + boot IFS |
| P-472 | MH2p executables are PIE (`ET_DYN`), `dio_manager` is `BIND_NOW`; only version need `libsocket.so.2`, defined by MU1329 `libsocket.so.3` | Static | Proven (PIE loading on 6.5: open) | `readelf -h -d -V`; `.gnu.version_r`; MU1329 `libsocket.so.3` `DT_VERDEF` |
| P-473 | NEEDED availability: `libcpp-ne.so.5`, `libopus`, NVIDIA libs, `libesoiap2`, `libpng14` absent on MU1329; ESO framework present with older ABI | Static | Proven | MH2p `DT_NEEDED`; MU1329 app.img listing, eifs/ifs2 dumps |
| P-474 | Unresolved symbol counts: libairplay 97, dio_manager 215, iap2connectionmanager 75, libEsoIAPTransport 44, libesoiap2 32, smartphone_integrator 68, libopus 1 | Static | Proven | pyelftools `.dynsym`/`PT_DYNAMIC` resolution against MU1329 libraries |
| P-475 | libc-level gap is only `quick_exit`, `__aeabi_l2f`, `__aeabi_idiv0/ldiv0` | Static | Proven (names) / Inferred (ABI) | same method, MU1329 `libc.so.3` (560,715 bytes, mifs), `libm.so.2`, `libsocket.so.3` |
| P-476 | ESO framework ABI differs (`libiplcommon` 22, `libcomm` 3, `libutil` 2, `libdsicommon` 1 missing for `dio_manager`); two C++ runtimes would collide | Static + Inference | Proven (symbols) / Inferred (collision) | as P-474 |
| P-477 | 320.17.1 video needs NvMedia/parser → `libnvtvmr`, `libnvrm`, `NvOs*` (TK1 driver stack) | Static | Proven | `libairplay.so` imports; `stage2_ifs3/lib/libnvmedia.so`, `app/armle/lib/libnvparser.so` imports |
| P-478 | MH2p Cinemo set is self-consistent on MU1329 names, needs `libpng14.so.0`, exports 49/50 SDK symbols MU1329 `dio_manager` imports (missing `CINEMO_OPTION_IAP_AUTHENTICATION`), and carries the wireless iAP2 components MU1329's lacks | Static | Proven (names) / Inferred (runtime) | `app/armle/usr/lib/libNme*.so`, `cinemo/libNmeVfs.so`; MU1329 counterparts; both `dio_manager.json` `iap2.cinemoLib` |

Regenerate after editing a subsystem table: the rows are copied verbatim from each document's
`## Evidence` section (109 entries on 2026-09-28).
