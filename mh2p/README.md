# MH2p Reference Analysis

MHI2 has no built-in wireless CarPlay, so its firmware alone cannot show how the pieces are meant to fit
together. The newer **MH2p** (MIB2 High Plus, `MH2p_ER_AUG33_P2873`) **ships** wireless CarPlay and has a
closely related software stack. This folder records what the MH2p firmware shows, as reference evidence
for the questions in [STATUS.md](../STATUS.md).

**Evidence rules for this folder**

- All findings are static: configuration, strings, symbols, disassembly and decompilation. No MH2p unit was
  observed at runtime, and nothing here was run on any car.
- Evidence IDs are `P-100`..`P-499`, kept separate from the MHI2 register (E-xxx). An MH2p fact is
  **cross-platform evidence**. It shows how a production unit does something; it is not proof of MHI2 behaviour.
- Comparisons with **MHI2Q MU1329** (`MHI2Q_ER_AUG22_P5092`) were checked against that firmware.
  MU1329 uses the same Cinemo / `dio_manager` / `smartphone_integrator` stack as MH2p. MU0678 facts come
  from this repository's own documents.

> **Disclaimer.** This is independent interoperability research on hardware the author owns. It is not
> affiliated with or endorsed by Audi, Volkswagen, Alpine, Cinemo or Apple; product names are used only
> to identify the systems described. No firmware, binaries or keys are distributed here: only short
> excerpts needed to document behaviour, interfaces and configuration values. Nothing in this folder has
> been tested on a car. Modifying a head unit can damage it; any use of this information is at your own
> risk.

## Paths

The firmware is not part of this repository. Paths in these documents use these placeholders:

| Placeholder | Meaning |
| --- | --- |
| `<MH2p SWDL>/` | The stock MH2p update package (`Meta/`, `Data/...`) |
| `<MH2p>/` | That package extracted, as described in [firmware.md](firmware.md): `app/` = `/mnt/app`, `efs-system/` = `/mnt/system`, `stage2_ifs0..5/` = the six boot IFS images, `stage2_images/` = those images as files |
| `<MU1329>/` | The MHI2Q MU1329 firmware (`MHI2Q_ER_AUG22_P5092`) extracted the same way: `app/` = `/mnt/app`, `system/` = `/mnt/system`, `ifs/` = boot IFS contents |

Values that could identify a unit or look like real credentials are masked with `****`. For example, the
sample Wi-Fi SSID and passphrase hard-coded in a test job.

## Reading order

| # | Document | Content |
| --- | --- | --- |
| 1 | [wireless-carplay-architecture.md](wireless-carplay-architecture.md) | End-to-end MH2p sequence, boot to session, 18 steps with evidence |
| 2 | [answers.md](answers.md) | STATUS.md Q1-Q13, roadmap items, traces and E-entries, answered from MH2p |
| 3 | [requirements.md](requirements.md) | Checklist: what MHI2 / MHI2Q would need, MH2p reference vs MU1329 vs MU0678 |
| 4 | [configuration-and-boot.md](configuration-and-boot.md) | `smartphone_integrator.json`, `dio_manager.json`, `iap2connectionmanager.json`, `connectivity.json`, boot scripts, mdnsd, pf, what starts what (P-100..P-123) |
| 5 | [wifi.md](wifi.md) | BCM4359, the 5 GHz CarPlay AP, hostapd templates, channels, Apple vendor IE, credential path, DHCP, firewall (P-200..P-222) |
| 6 | [bluetooth-iap2.md](bluetooth-iap2.md) | `btstack` RFCOMM iAP2 server, `/dev/iapDevice-*`, `iap2connectionmanager`, Cinemo transport plug-ins, iAP2 over AirPlay (P-300..P-326) |
| 7 | [airplay-dio.md](airplay-dio.md) | `libairplay` 320.17.1 vs 210.81, HomeKit pairing, `_carplay-ctrl` trigger, `interfaceName`, BT-MAC match, coding gate (P-400..P-426) |
| 8 | [portability.md](portability.md) | Can MH2p binaries run on MU1329 / MU0678? Per-binary verdicts (P-470..P-478) |
| 9 | [binaries.md](binaries.md) | Inventory with hashes; presence across MH2p / MU1329 / MU0678 |
| 10 | [firmware.md](firmware.md) | Provenance, hashes, partition formats, how it was extracted |
| 11 | [evidence.md](evidence.md) | All 109 `P-` entries in one register |

## Findings in brief

1. **The Bluetooth bootstrap is a plain RFCOMM byte stream.**
   - `btstack` registers the iAP2 SDP record (UUID `00000000-deca-fade-deca-deafdecacaff`) and checks the
     detect bytes `FF 55 02 00 EE 10`.
   - It then exposes the link as `/dev/iapDevice-<BT address>`, with no iAP2-specific driver ABI.
   - `enableIap` is `true` on MH2p. It is the `bluetooth` app's iAP2 policy flag, not the RFCOMM server
     switch.
2. **The bootstrap runs in its own small process, not in DIO.** `iap2connectionmanager`:
   - opens that node and identifies Bluetooth (99) and Wireless CarPlay (98) transport components;
   - answers `RequestAccessoryWiFiConfigurationInformation` with the SSID, passphrase, security and channel
     of the 5 GHz AP.
3. **The phone is found by Bonjour and matched by its Bluetooth MAC.**
   - `smartphone_integrator` browses the phone's `_carplay-ctrl._tcp` on the AP interface and starts
     `dio_manager` for a Wi-Fi device.
   - Inside `dio_manager`, `libairplay`'s `CarPlayControlClient` sends `GET /ctrl-int/1/connect` only to
     the controller whose device ID equals the phone's BT MAC.
   - No iAP2 0x4300/0x4301 names appear in MH2p.
4. **Wired and wireless use the same `dio_manager`.**
   - The AirPlay `interfaceName` is `carplay0` for USB, or the AP interface for Wi-Fi.
   - `mdnsd` runs at boot on all interfaces.
   - After the session starts, iAP2 continues inside AirPlay as the `iAPSendMessage` command.
5. **The Wi-Fi side is ordinary.**
   - A dedicated 5 GHz WPA2 AP (`bcm0`, channel 36 or 149, 11ac).
   - `dnsmasq`, and pf holes for the CarPlay ports and mDNS.
   - One Apple vendor IE (OUI `00:a0:40`) that carries the head unit's BT MAC.
   - Wireless CarPlay is refused without the 5 GHz AP.
6. **The receiver is AirPlay 320.17.1.** Relative to MHI2Q's 210.81 it adds:
   - HomeKit pair-setup/verify;
   - the `_carplay-ctrl` trigger;
   - iAP2 over AirPlay;
   - Opus audio.

   It is **not portable** to MU1329: it needs NVIDIA Tegra K1 video (NvMedia), `libcpp-ne.so.5` and a
   newer ESO framework. MU1329 turned out to be Qualcomm APQ8064 / QNX 6.5.0 SP1; MH2p is Tegra K1 / QNX 6.6.0.
7. **For MHI2Q, the missing pieces are not configuration.**
   - The iAP2 RFCOMM server in `btstack` is missing.
   - The bootstrap iAP2 client is missing.
   - A HomeKit-pairing AirPlay receiver with the `_carplay-ctrl` trigger and `iAPSendMessage` is missing.
   - The AP work (5 GHz, IE, pf, mDNS) is configuration, apart from the open question of whether the 88W8787
     works on 5 GHz. See [requirements.md](requirements.md).

## Still open (across the folder)

- Whether the Bluetooth iAP2 link stays up during the Wi-Fi session, and what `disableBluetooth` closes.
- The 0x5703 byte encoding inside `libesoiap2.so`, and how the AP SSID/passphrase are generated and persisted.
- Which coding value feeds `dio_manager`'s persistence key 8877.
- Callers of `libairplay`'s packet/multicast socket helpers (STATUS Q6).
- Whether MU0678's `btstack` and `libairplay.so` contain any of the MH2p pieces. This is the cheapest next
  check; see [requirements.md](requirements.md).
