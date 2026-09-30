# What Wireless CarPlay on MHI2 / MHI2Q Would Need (MH2p as the Reference)

This turns the MH2p findings into a checklist. For each building block it gives the MH2p reference value
(proven on MH2p), what MHI2Q MU1329 and MHI2 MU0678 have today, and what is missing.

**Status of this document: inference.** The MH2p column is evidence. The MU1329 column was checked against
the MU1329 firmware. The MU0678 column is taken from this repository's own docs.
The "missing" column is a design inference, and **nothing here has been built or run on a car**.

## Checklist

| # | Building block | MH2p reference | MHI2Q MU1329 today | MHI2 MU0678 (per repo docs) | Missing for MHI2Q |
| --- | --- | --- | --- | --- | --- |
| 1 | **5 GHz access point** | BCM4359 `bcm0`, `hw_mode=a`, HT40+, 11ac (80 MHz off), channel 36 or 149 per country (non-DFS), WPA2-PSK CCMP, max 8 stations. SI refuses wireless CarPlay without it (P-205..P-207, P-214) | Marvell 88W8787 `uap0`, 2.4 GHz as configured. `uaputl` accepts `sys_cfg_channel_ext` with the 5 GHz band ([wifi](wifi.md) §6) | Marvell 8787 `uap0` (E-001, E-002) | A 5 GHz configuration of `uap0`. **Whether the 88W8787 module and antenna work on 5 GHz is the open hardware question.** |
| 2 | **Apple vendor IE in the beacon** | `wl add_ie 7 … 00:a0:40 …`: OUI type 0, flags TLV (bit 0x01 = 5 GHz), name `MIB2P`, model `A8`, manufacturer `Audi AG`, BT MAC (P-210, P-211) | None. `uaputl sys_cfg_custom_ie` exists ([wifi](wifi.md) §6) | not documented | Add the IE with `sys_cfg_custom_ie` using the MH2p/LIVI layout |
| 3 | **Firewall** | pf opens TCP `5000:5001, 5010, 6000:6001, 6030, 6100, 6200, 7000:7001, 7100`, UDP `5001, 5020, 5353, 6000:6003, 6010:6011, 6020:6021, 7010:7011, 7070:7071`, mDNS v4/v6 and ICMPv6 on the AP (P-113, P-220) | `pf.conf` passes none of these on `uap0` ([configuration-and-boot](configuration-and-boot.md) §6) | not documented | The same rules on `uap0` |
| 4 | **DHCP on the AP** | `dnsmasq -i bcm0`, 10.174.189.128-247, 3 days, no router without uplink (P-218) | `dnsmasq` present (`eso/bin/apps/dnsmasq`); its `uap0` range not checked here | `uap0` 10.173.189.1/24, DHCP 10.173.189.10-99 ([docs/wifi.md](../docs/wifi.md)) | Probably nothing |
| 5 | **mDNS on the AP** | `mdnsd` at boot on all interfaces; `MDNS_DIRECTLINK_IFACE=carplay0` stays (P-111, P-219) | `mdnsd` per wired session, spawned by `dio_manager` (P-111) | `mdnsd` consumes the variable (E-029); producer unresolved (E-031) | An `mdnsd` instance that covers `uap0` when the phone joins |
| 6 | **Bluetooth iAP2 RFCOMM server** | `btstack`: SDP record with UUID `00000000-deca-fade-deca-deafdecacaff` on RFCOMM, detect `FF 55 02 00 EE 10`, byte-stream node `/dev/iapDevice-<addr>` (P-301, P-303) | **Absent.** `btstack` has no iAP SDP record, UUIDs, detect check or node (P-321) | `iap`/`CIapBTChannel` and a BT iAP proxy exist (E-014, E-015, E-021); `btstack` side not documented | The whole piece. This is the **make-or-break item** for MHI2Q: either another process registers an RFCOMM server through the Blue SDK (not established), or `btstack` is extended |
| 7 | **Bootstrap iAP2 client over Bluetooth** | `iap2connectionmanager` (ESO `libesoiap2`): link parameters `bt` 5/2048/3000/700/30/3, identifies BluetoothTransportComponent 99 (+ BT MAC) and WirelessCarPlayTransportComponent 98, MFi auth, receives `WirelessCarPlayUpdate` / `DeviceTransportIdentifierNotification`, answers 0x5702 with 0x5703 (P-306..P-311) | **Absent.** Cinemo in `dio_manager` identifies a Bluetooth component only, with no Wi-Fi/OOB/0x5703 responders (P-322) | `ipod-drvr-iap2.so` has compiled BT/Wi-Fi components and a 0x5703 path (E-033..E-041), activation unproven | A small separate client. The MU1329 Cinemo transport-plugin hook could carry the link but not the wireless messages (P-319, P-320) |
| 8 | **Wi-Fi credentials source** | `connectionmanager` → SI `WiFiAPInfoProvider` → 0x5703 {SSID, passphrase, security, channel} (P-215, P-309) | Nothing wired to iAP2 | E-041 (response fields) | Read the `uap0` configuration and answer 0x5702 |
| 9 | **Discovery and session trigger** | SI browses `_carplay-ctrl._tcp` on the AP; `libairplay` `CarPlayControlClient` sends `GET /ctrl-int/1/connect` with `AirPlay-Receiver-Device-ID` to the controller whose device ID = the phone's BT MAC (P-313, P-314, P-406, P-407, P-422) | **Absent.** 210.81 has no `CarPlayControl` code (checked) | not documented | A controller browse + connect, or the alternative iAP2 0x4300/0x4301 flow used by other receivers |
| 10 | **AirPlay receiver with HomeKit pairing** | 320.17.1: `/pair-setup` (SRP-3072) + `/pair-verify` (Ed25519, Curve25519, ChaCha20-Poly1305, HKDF-SHA512), keychain on `/mnt/misc1/carplay` (P-403) | **Absent.** 210.81 is MFi `/auth-setup` only (P-403) | not documented | A different receiver. 320.17.1 is not portable to MU1329: NVIDIA NvMedia video, `libcpp-ne.so.5`, newer ESO framework (P-474..P-477) |
| 11 | **Bind the receiver to the AP** | `interfaceName` = AP interface; `_airplay._tcp` registered by `if_nametoindex` (P-404, P-420) | Mechanism exists, value hard-coded `carplay0` (P-423) | Same mechanism (E-026) | Only the value. Trivial compared with 9, 10, 12 |
| 12 | **iAP2 during the session** | AirPlay command `iAPSendMessage` both ways; `CIAP2ServiceEso` in `dio_manager`; no stream 130 in this build (P-315, P-316, P-411, P-412) | **Absent.** No `iAPSendMessage` (P-424) | not documented | Part of the new receiver. `dio_manager`'s USB-bound Cinemo client does not receive this |
| 13 | **Wireless audio** | Opus 16/24/48 kHz mono, plus AAC-ELD/AAC-LC/PCM (P-413); `*AudioWireless` fragment sizes (P-414) | No Opus. `libopus.so` from MH2p is a library-level drop-in (P-474, [portability](portability.md) §1) | not documented | Opus in the receiver |
| 14 | **Coding / enablement** | SI `carplayWireless`; `dio_manager` persistence key 8877 byte 0x17 bit 3 (P-105, P-425) | No CarPlay-technology coding in `dio_manager` (P-425) | not documented | Not needed if the wireless path is separate |
| 15 | **Phone correlation** | BT MAC in the Apple IE, in iAP2 component 99, and as the `_carplay-ctrl` match key (P-211, P-307, P-422) | n/a | TRACE-006 open | Use the same key |

## What MH2p settles

- **The architecture.** Keep the wired stack unchanged. Add three pieces:
  - a Bluetooth iAP2 bootstrap (items 6-8);
  - Wi-Fi AP settings (items 1-5);
  - a wireless AirPlay session path (items 9-13).

  MH2p itself uses the same `dio_manager` for both, but its binaries are not portable ([portability.md](portability.md)).
- **What does not work.** Redirecting DIO's USB iAP2 client (`/dev/ipod0` on MU0678, `/dev/otg-cinemo` on MU1329)
  to Bluetooth is not how production does it. The 0x9999 client ABI is not used on the Bluetooth node
  ([answers.md](answers.md) Q12, Q13).
- **The AP side is ordinary configuration.** A 5 GHz WPA2 AP, dnsmasq, a global mDNS, pf holes and one
  vendor IE. The only open item there is the 88W8787's 5 GHz RF path.
- **The Bluetooth side is new code on MHI2Q.** Its `btstack` lacks the iAP2 RFCOMM server entirely.

## Cheapest next checks (read-only, suggested order)

These are checks to run on the MHI2/MHI2Q firmware or on a bench, not changes to a car.

1. **MU0678 `btstack`.** Search for the iAP2 UUID bytes `00000000decafadedecadeafdecacaff`, the reversed
   `ffcacaced…` form, `FF 55 02 00 EE 10` handling, `/dev/iapDevice-` and `updIapDevicePath`. A hit would
   mean MU0678 already has item 6 (MU1329 does not).
2. **MU0678 `libairplay.so`.** Search for `CarPlayControl`, `pair-verify`, `iAPSendMessage`, `OPUS`. This
   tells whether items 9, 10 and 12 exist there.
3. **5 GHz on the 88W8787.** Read `uaputl sys_cfg_scan_channels` / the supported band list on a bench
   unit before planning item 1.
4. **The MU1329 Blue SDK.** Find out whether any process other than `btstack` can register an RFCOMM
   server and SDP record.
