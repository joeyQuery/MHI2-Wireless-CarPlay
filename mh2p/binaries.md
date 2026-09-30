# MH2p Binary Inventory (wireless CarPlay relevant)

Baseline: [firmware.md](firmware.md). Paths are relative to the partition mount point (`/mnt/app` for
`eso/`, `armle/`; the IFS for `lib/dll/` entries marked *IFS*). Hash = first 16 hex digits of SHA-256.

> Presence proves availability in the image, not participation in the wireless path. Roles are taken from
> the subsystem documents; see those for the evidence.

## Wireless CarPlay chain

| Binary | Size | SHA-256 | Role (see) |
| --- | --- | --- | --- |
| `eso/bin/apps/smartphone_integrator` | 1,090,576 | `4311bef1b6fc0933…` | Starts and supervises the CarPlay / Android Auto children ([configuration-and-boot](configuration-and-boot.md)) |
| `eso/bin/apps/dio_manager` | 958,480 | `f1288c262a03f6b1…` | CarPlay session owner: iAP2 (Cinemo), AirPlay server, audio/HID ([airplay-dio](airplay-dio.md)) |
| `eso/lib/libairplay.so` | 1,024,064 | `9047820a1ab598ad…` | Apple AirPlay receiver, `AirPlay/320.17.1` ([airplay-dio](airplay-dio.md)) |
| `eso/lib/libesoiap2.so` | 376,176 | `32557ead406153aa…` | ESO iAP2 wrapper used by `dio_manager` ([bluetooth-iap2](bluetooth-iap2.md)) |
| `armle/usr/lib/libNmeSDK.so` | 1,192,374 | `b33912971e75c010…` | Cinemo NME SDK ([bluetooth-iap2](bluetooth-iap2.md)) |
| `armle/usr/lib/cinemo/libEsoIAPTransport.so` | 100,892 | `83eced1ff7d7909d…` | ESO-provided iAP transport plug-in for Cinemo ([bluetooth-iap2](bluetooth-iap2.md)) |
| `armle/usr/lib/cinemo/libNmeAppleAuth.so` (+ `libNmeAppleAuthImpl.so`) | 87,728 | `742b0c6d017486cd…` | Apple authentication for Cinemo |
| `eso/bin/apps/iap2connectionmanager` | 297,164 | `dbe8882fc9b0b2ce…` | iAP2 connection manager ([bluetooth-iap2](bluetooth-iap2.md)) |
| `eso/bin/apps/iap` | 182,020 | `3722f4c76720dd23…` | iAP application (MU0678 also has one) |
| `eso/bin/apps/bluetooth` | 1,255,364 | `eeefaaec816bbab3…` | Bluetooth service ([bluetooth-iap2](bluetooth-iap2.md)) |
| `eso/bin/apps/btstack` | 2,071,484 | `89f93f2e2f7f78d8…` | Bluetooth protocol stack |
| `eso/bin/apps/connectionmanager` | 1,154,972 | `0c9b740d982ff62e…` | WLAN station/AP orchestration ([wifi](wifi.md)) |
| `eso/bin/apps/connectivity_launcher` | 223,072 | `7dc0d19cf253e329…` | Connectivity process launcher |
| `armle/usr/sbin/hostapd-2.5-wapi` | 840,788 | `7e0b78aae62d9568…` | Wi-Fi access point daemon ([wifi](wifi.md)) |
| `armle/usr/sbin/wpa_supplicant-2.5-wapi` | 1,260,335 | `8f45ebd0609f0993…` | Wi-Fi station daemon |
| `armle/usr/sbin/wl` | 358,743 | `e23e95f2ec1561be…` | Broadcom `wl` utility |
| *IFS* `lib/dll/devnp-qwdi-2.5_bcm4359-wapi.so` | 490,260 | `1d6d3bd497cd40b4…` | io-pkt driver for the BCM4359 ([wifi](wifi.md)) |
| `armle/usr/sbin/mdnsd` | 440,824 | `23fb76524071a317…` | Bonjour daemon |
| `armle/lib/dll/nss_mdnsd.so` | 18,990 | `01b3fa41e226243e…` | NSS plug-in for mDNS names |
| `eso/lib/libopus.so` | 232,208 | `a4f11cefe74e4657…` | Opus codec (wireless CarPlay audio) |
| `eso/bin/apps/carplay_colors` | 55,248 | `39a7c4bfb8a61cef…` | CarPlay theme colours |

## USB CarPlay path (for comparison)

| Binary | Size | SHA-256 |
| --- | --- | --- |
| `armle/lib/dll/devnp-ncm.so` | 54,037 | `47d1abd375e2a1b7…` |
| `armle/lib/dll/devnp-usbdnet.so` | 92,227 | `f406b30e009c5d8d…` |
| `armle/lib/dll/devu-iap2-tegra3-ci.so` | 35,042 | `3289005d1b9fb62e…` |
| `armle/lib/dll/devu-iap2ncm-tegra3-ci.so` | 35,214 | `12d2dc21d9f52b75…` |

The `tegra3` device-controller names are carried over from the MIB2 generation even though MH2p is a
Tegra K1.

## Presence compared with MHI2Q MU1329 and MHI2 MU0678

MU1329 = full listing of the stock `app.img` (`MHI2Q_ER_AUG22_P5092`). MU0678 = the component list in
[`docs/firmware.md`](../docs/firmware.md) and [`docs/binaries.md`](../docs/binaries.md).

| Component | MH2p | MHI2Q MU1329 | MHI2 MU0678 |
| --- | --- | --- | --- |
| `dio_manager`, `smartphone_integrator` | yes | yes | `dio_manager` yes |
| `libairplay.so` | 320.17.1 | 210.81 | yes (version not recorded) |
| Cinemo `libNmeSDK.so` | yes | yes | not listed |
| `libEsoIAPTransport.so`, `libesoiap2.so` | yes | **no** | not listed |
| `libNmeAppleAuth*.so` | yes | **no** | not listed |
| `iap2connectionmanager` | yes | **no** | not listed |
| `iap` | yes | **no** | yes |
| `ipod-drvr-iap2.so`, `libiap2client.so` | no | no | yes |
| `libopus.so` | yes | **no** | not listed |
| `hostapd` / `wpa_supplicant` | 2.5-wapi | yes (`armle/sbin/`) | not listed |
| Marvell `uaputl` | no | yes | yes |
| Broadcom `wl` / BCM4359 driver | yes | no | no |
| `devu-iap2*-tegra3-ci.so` | yes | not in app.img | yes |

What MU1329 lacks is exactly the set that carries the wireless path on MH2p: the extra iAP transport
plug-in, `iap2connectionmanager`, the Apple authentication plug-in, Opus, and the 320-series AirPlay
receiver. Whether these can run on MU1329 is covered in [portability.md](portability.md).
