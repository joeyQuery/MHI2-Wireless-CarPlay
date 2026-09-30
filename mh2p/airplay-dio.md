# MH2p AirPlay Receiver and DIO Session Handling for Wireless CarPlay

How the MH2p (`MH2p_ER_AUG33_P2873`, `MMX2P` 2201.8.0) AirPlay library and `dio_manager` run a wireless
CarPlay session, compared with the MHI2Q MU1329 stack and, where the MHI2 repo makes a claim, with MU0678.

> **Scope rule.** Sections marked "MH2p" are static evidence from the MH2p firmware (see
> [firmware.md](firmware.md)). They are **cross-platform evidence**, not proof of MHI2 / MHI2Q behaviour.
> Statements about MHI2 / MHI2Q are inference unless they cite an MU1329 file. Evidence IDs P-400..P-469
> are listed at the end.

Paths are relative to `<MH2p>/` unless stated. MU1329 paths are relative to
`<MU1329>/` (see [README](README.md#paths)). **Addresses are Ghidra addresses** (image base 0x10000 for the
MH2p `ET_DYN` files and for MU1329 `libairplay.so`; subtract 0x10000 for `objdump` / file offsets). MU1329
`dio_manager` is `ET_EXEC`, its addresses are real virtual addresses. MH2p code is Thumb-2, so exported
symbol values are odd.

Related documents: [bluetooth-iap2.md](bluetooth-iap2.md) (P-300..), [wifi.md](wifi.md) (P-200..),
[configuration-and-boot.md](configuration-and-boot.md) (P-100..), [portability.md](portability.md) (P-470..).

---

## 1. Summary

| Question | MH2p answer | MU1329 (MHI2Q) |
| --- | --- | --- |
| AirPlay library | `eso/lib/libairplay.so`, `AirPlay/320.17.1`, GCC 4.9.2, Thumb-2 + NEON | `AirPlay` 210.81, GCC 4.4.2, ARM, VFPv3-D16 |
| Pairing | HomeKit-style `/pair-setup` + `/pair-verify` (SRP-3072/SHA-512, Ed25519, Curve25519, ChaCha20-Poly1305, HKDF-SHA512), keychain on `/mnt/misc1/carplay` | None (no `pair-setup`/`pair-verify`), MFi `/auth-setup` only |
| Streams accepted in SETUP | 100, 101, 102 (audio), 110 (screen). Everything else is logged "Unsupported stream type" | Same family of audio/screen streams (not decompiled in detail) |
| iAP2 over Wi-Fi | AirPlay **command** `iAPSendMessage` in both directions; no stream 130 | No `iAPSendMessage` anywhere |
| Wireless audio | `OPUS/16000/1`, `OPUS/24000/1`, `OPUS/48000/1` + AAC-LC, AAC-ELD, PCM | ALAC, AAC-LC, AAC-ELD, PCM; no Opus |
| Head-unit Bonjour | Registers `_airplay._tcp` only; **browses** the phone's `_carplay-ctrl._tcp` (both in `libairplay` inside `dio_manager`, and in `smartphone_integrator`) | Registers `_airplay._tcp` only; no `_carplay-ctrl` code |
| Session trigger | `CarPlayControlClient` sends `GET /ctrl-int/1/connect` with `AirPlay-Receiver-Device-ID` to the phone whose Bluetooth MAC matches | n/a |
| `interfaceName` | Set by `dio_manager` right after `AirPlayReceiverServerCreate`: `carplay0` (USB) or the interface that owns `wifi.ipaddr` 10.174.189.1 (Wi-Fi), fallback `bcm0`/`uap0` | Set by `dio_manager` to the constant `carplay0` |
| Timing | NTP-style timing channel (`timingPort`, `_TimingNegotiate`); no PTP | NTP (`NTPClock*`, `AirPlayNTPClient_*`); no PTP |
| Coding gate | `DIOCodingProvider` derives `enCarPlayTechnology` (USB_ONLY / WIFI_ONLY / BOTH) from persistence key 8877 | No CarPlay-technology coding in `dio_manager` |

---

## 2. libairplay.so: 320.17.1 (MH2p) vs 210.81 (MU1329)

### 2.1 Identity and build

- MH2p `app/eso/lib/libairplay.so` (1,024,064 bytes): version string `AirPlay/320.17.1` at file offset
  0xc8094, and `srcvers` TXT value `"320.17.1"` written by `_UpdateBonjourAirPlay` (§2.4). `.comment`:
  `GCC: (GNU) 4.9.2` and `4.7.3`. Build RPATH `.../MIB2P_build_docker_CLU28_newPDK/...`. **[P-400]**
- NEEDED: `libbacktrace.so.1 libasound.so.2 libopus.so libnvmedia.so libnvparser.so libdisplayinit.so
  libscreen.so.1 libdns_sd.so.1 libsocket.so.3 libm.so.2 libnbutil.so.1 libcpp-ne.so.5 libc.so.3`.
- MU1329 `app/eso/lib/libairplay.so`: `210.81` (0x9df2c), GCC 4.4.2; NEEDED has `libomxctx.so.1
  libOmxBase.so libOmxCore.so libOSAbstraction.so libecpp-ne.so.4` instead of Opus/NVIDIA/`libcpp-ne.so.5`.
  ELF details are in [portability.md](portability.md).

### 2.2 Exported API difference

`readelf --dyn-syms` (via pyelftools) on both files: 1,914 defined global symbols in 320.17.1, 1,468 in
210.81. **[P-401]**

Present only in 320.17.1 (non-C++ names, selection):

| Family | Symbols |
| --- | --- |
| CarPlay control client (wireless trigger) | `CarPlayControlClientCreateWithServer`, `…Start`, `…Stop`, `…Connect`, `…Disconnect`, `…STALeft`, `CarPlayControllerCopyName`, `CarPlayControllerGetBluetoothMacAddress`, `CarPlayControllerGetInterfaceName`, `CarPlayControllerCopySourceVersion`, `BonjourBrowser_*`, `BonjourDevice_*` |
| HomeKit pairing | `PairingSession*` (Create, Exchange, DeriveKey, SavePeer, FindPeer, SetSetupCode, …), `AirPlayCopyHomeKitPairingIdentity`, `AirPlayReceiverSessionSetHomeKitSecurityContext`, `SRP*`/`SRP6a_*`, `kSRPParameters_3072_SHA512`, `ed25519_*_ref`, `curve25519_donna`, `chacha20_poly1305_*`, `poly1305*`, `HKDF_SHA512_compat`, `HMAC_SHA512*`, `Keychain*`, `SecItem*_compat`, `mp_*` (LibTomMath) |
| Session / screen | `AirPlayReceiverSessionSendiAPMessage`, `AirPlayReceiverSessionUpdateVehicleInformation`, `AirPlayReceiverSessionSendHIDReport`, `AirPlayReceiverSessionFlushAudio`, `AirPlayReceiverSessionScreen_SetChaChaSecurityInfo`, `AirPlayReceiverServerSetMaxFps`, `AirPlayReceiverServerSetWindowsOffset`, `AirPlayReceiverServerStopAllAudioConnections`, `ScreenStreamSetMaxFPS`, `ScreenStreamSetAVCC`, `NetTransportChaCha20Poly1305Configure`, `AudioStreamSetTransportType` |
| MFi | `MFiSAP_*`, `MFiPlatform_*`, log category `gLogCategory_MFiQNX` |
| Codecs | `AudioConverter*` (non-`_compat`), log categories `OPUSCodec`, `AudioCodecCinemo`, `NvMediaVideoDecoder*`, `QNXScreenRenderDM` |

Present only in 210.81 (selection): `AirPlayNTPClient_*`, `NTPClock*`, `AirTunesServer_*`,
`HIDBrowser*`/`HIDDevice*`, `APSMFiSAP_*`/`APSMFiPlatform_*`, `AirPlaySettings_*`, the
`AirPlayReceiverSessionScreen_Set*` setters (`SetIFName`, `SetTransportType`, `SetClientIfMACAddr`,
`SetClientDeviceID`, …), `IsWiFiNetworkInterface`, `IsUSBNetworkInterface`, log category
`OMXVideoDecoder`. C++ symbols: 568 only in 320.17.1, 270 only in 210.81.

The MU1329 screen setters `AirPlayReceiverSessionScreen_SetIFName` / `…SetTransportType` /
`…SetClientIfMACAddr` have `st_size` 4 (one `bx lr`), i.e. the same no-op stubs the MHI2 repo found in
MU0678 (E-018). In 320.17.1 they are gone entirely. **[P-402]**

### 2.3 Pairing and authentication

MH2p (strings in `libairplay.so`, offsets = file offsets): **[P-403]**

- HTTP endpoints `/pair-setup` (0xc3014), `/pair-verify` (0xc3074), `/auth-setup`, `/info`, `/command`,
  `/feedback`, `/diag-info`, `/logs`; headers `X-Apple-HKP` (0xc3020), `X-Apple-PD` (0xc309c).
- Server-side pair-setup M1..M6 and pair-verify M1..M4 log strings (0xda1a8..0xda458, 0xd9a10..0xd9d80),
  HKDF labels `Pair-Setup-Encrypt-Salt/Info`, `Pair-Verify-ECDH-Salt/Info`, `Pair-Verify-Encrypt-*`,
  `Control-Salt`, `Control-Read/Write-Encryption-Key`, `Events-Salt`, `DataStream-Salt`,
  `MFi-Pair-Setup-Salt/Info`, "Pair-setup server disabled after too many attempts", throttling.
- Keychain file `/mnt/misc1/carplay/Library/KeyChains/carplay.keychain` (0xd84e0). The
  `smartphone_integrator.json` `paths.cleanupPaths` `["/mnt/misc1/carplay"]` matches it (factory reset of
  the pairing store is an inference).
- The `_airplay._tcp` TXT gets `pi` = `AirPlayCopyHomeKitPairingIdentity()` (§2.4).
- MFi: `MFiSAP_*` / `MFiPlatform_*`; the MFi chip is configured outside `dio_manager.json` (`/dev/i2cmfi:34`,
  P-121).

MU1329 210.81: 0 occurrences of `pair-setup`, `pair-verify`, `X-Apple-HKP`, `DataStream-Salt`,
`iAPSendMessage`. Only `/info`, `/auth-setup` and `APSMFiSAP_*`.

xcertplay (the open-source receiver, `shilapi/xcertplay`) runs `pair-setup` (first
time) and `pair-verify` for a wireless session. That the iPhone requires them over Wi-Fi is inference from
xcertplay plus the MH2p code; MH2p shows a production receiver that implements them.

### 2.4 Bonjour: what the receiver registers and browses

**Registration** (`_UpdateBonjourAirPlay`, MH2p `FUN_0003e0dc`, 1,230 bytes; callers `FUN_0003e640` and
`AirPlayCopyServerInfo`): **[P-404]**

- TXT keys in order: `deviceid` (= `HardwareAddressToCString(obj+0x19b, 6)`), `features` (`0x%X` or
  `0x%X,0x%X` from `AirPlayGetFeatures`), `fv` (optional), `flags` (`AirPlayGetStatusFlags`, if non-zero),
  `model` (obj+0x80), `pi` (HomeKit pairing identity, if available), `srcvers` = `"320.17.1"`. No `pk`,
  no `protovers` (xcertplay publishes both).
- `DNSServiceRegister(obj+0x7c, flags 0x800, ifIndex, name, "_airplay._tcp.", "local.", …, port, TXT)`
  where `ifIndex = if_nametoindex(obj+0x1a1)` when obj+0x1a1 is non-empty, else 0.
- MU1329 210.81 (`FUN_00030fb8`): same structure, `ifIndex = if_nametoindex(obj+0x16c)`, flags 0.
  The MHI2 repo reports `interfaceName` at obj+0x6c for MU0678 (E-026); the offset differs per build,
  the mechanism is the same.
- The strings `_raop._tcp.`, `_hap._tcp.`, `_mfi-config._tcp.`, `_airport._tcp.` exist in 320.17.1 but are
  referenced only by `BonjourBrowser`/`AsyncConnection` utility and self-test functions
  (`FUN_0005fe74`, `FUN_00060534`, `FUN_00060380`, `BonjourBrowser_Test`, `AsyncConnection_Test`). No
  second registration was found. **[P-405]** (Partial: no exhaustive call-site scan of
  `DNSServiceRegister`.)

**Browsing** (`_CarPlayControlClientEnsureStarted`, `FUN_0004f9f4`): `BonjourBrowser_Create(...,
"CarPlayControlClient")`, `BonjourBrowser_Start(browser, "_carplay-ctrl._tcp", "local.", 0, 0, 0x8000000)`.
If no device ID is set yet it uses `AirPlayGetDeviceID()`. **[P-406]**

**Connect** (`_CarPlayControlClientSendCommand`, `FUN_00050330`): **[P-407]**

- Resolves the controller address, formats it as `"%s%%%u"` (address with scope ID, i.e. IPv6 link-local
  capable), logs `"CarPlayControl connecting to iPhone: %s on port %d | wifi: %d"`.
- `HTTPClientCreate`, `HTTPClientSetTimeout(2)`, flags `0x104` when the service dictionary's `wifi` key is
  set, else `0x4`.
- Request `GET /ctrl-int/1/connect HTTP/1.1` (`snprintf("/ctrl-int/1/%s", "connect")`), headers `Host`
  and `AirPlay-Receiver-Device-ID` (0xc80a8).
- This matches xcertplay's `GET /ctrl-int/1/connect` (03-wireless.md §2).

`CarPlayControllerGetBluetoothMacAddress` (0x50bb0) returns `BonjourDevice_GetDeviceID()` of the
controller's Bonjour record. `CarPlayControllerGetInterfaceName` (0x50ca4) returns the `ifname` of the
first entry in the record's service array. **[P-408]**

MU1329 210.81 has no `_carplay-ctrl` string and no CarPlayControl code.

### 2.5 Server properties (`AirPlayReceiverServerSetProperty`)

MH2p (0x3e93c, 230 bytes); property names are CF-lite constant strings (layout `0x10756, 0x7fffffff,
inline chars`): **[P-409]**

| Property | Action |
| --- | --- |
| `deviceID` | `CFGetData(value, obj+0x19b, 6)` (raw 6-byte MAC used for TXT `deviceid`) |
| `interfaceName` | `CFGetCString(value, obj+0x1a1, 0x11)` (used by `_UpdateBonjourAirPlay`) |
| `playing` | bool at obj+0x198; may restart Bonjour |
| `model` | `CFGetCString(value, obj+0x80, 0x100)` |
| anything else | `AirPlayReceiverServerPlatformSetProperty` |

MU1329 210.81 (0x318d0): the same four keys, at obj+0x166 (`deviceID`), obj+0x16c (`interfaceName`),
obj+0x160 (`playing`), obj+0x60 (`model`).

### 2.6 Stream types

`AirPlayReceiverSessionSetup` (MH2p internal implementation at 0x47170, 7,840 bytes; the exported symbol
0x366fc is a PLT thunk) iterates the `streams` array and switches on `type`: **[P-410]**

| `type` | Handling |
| --- | --- |
| 100 (0x64), 101 (0x65) | Main / Alt audio (`_MainAltAudioSetup`, labels "Main"/"Alt") |
| 102 (0x66) | General audio (`_GeneralAudioSetup`, "main high") |
| 110 (0x6e) | Screen (`_ScreenSetup`) |
| default | `"<AirPlay> Unsupported stream type: %d"`, stream skipped |

- There is **no case for 111 (altScreen) or 130 (iAP2 data stream)**.
- The seed-based key derivation `FUN_00044dc4`
  (`HKDF("DataStream-Salt%llu", "DataStream-Output/Input-Encryption-Key")` via `PairingSessionDeriveKey`)
  is called from the 100/101, 102 and 110 branches (0x47424, 0x479d6, 0x4827c): per-stream keys come from
  the pair-verify session. Screen traffic can use ChaCha20-Poly1305
  (`AirPlayReceiverSessionScreen_SetChaChaSecurityInfo`, `NetTransportChaCha20Poly1305Configure`).
- An audio log line documents the numbering: `"Stream type: %d ( 0 - invalid, 100 - main, 101 -
  alternate, 102- main high)"` (0xeb7dc).

### 2.7 iAP2 over the AirPlay session

- `AirPlayReceiverSessionSendiAPMessage` is exported (HU → phone). **[P-411]**
- `dio_manager` `handleSessionControl` (`FUN_0008f3bc`, 984 bytes) compares the command name with CF
  constants resolved to `startSession`, `disableBluetooth`, `hidSetInputMode`, `iAPSendMessage`,
  `updateVocoderInfo`. For `iAPSendMessage` it takes the `data` CFData from the params, logs
  `"<iAP2> Received message: %p, size: %zu"`, copies it and passes it to the iAP2 transport object at
  `this+0x3b0` (phone → HU). This matches P-315/P-316.
- For `disableBluetooth` it reads a MAC string from the params, logs `"<BT> iPhone requires disconnection
  from BT stack, iPhone's MAC"` and calls `FUN_0008db20(this, 1, mac)`.
- MU1329 `dio_manager` has `disableBluetooth` (1 string) but no `iAPSendMessage`, `updateVocoderInfo`.

So in 320.17.1, iAP2 after the Wi-Fi handover rides in AirPlay commands, not in an AirPlay data stream.
xcertplay (a newer, 950.x sender stack) sees stream 130 instead (03-wireless.md §7-8). Which form a current
iPhone chooses for a 320.17.1 receiver is not determinable from firmware; that it works in production cars
is an inference from MH2p shipping it. **[P-412]**

### 2.8 Audio

- 320.17.1 format table (in `AirPlayReceiverSessionSetup`): PCM 8/16/24/32/44.1/48 kHz,
  `AAC-LC/44100/2`, `AAC-LC/48000/2`, `AAC-ELD/{44100,48000}/{1,2}`, `AAC-ELD/16000/1`, `AAC-ELD/24000/1`,
  **`OPUS/16000/1`, `OPUS/24000/1`, `OPUS/48000/1`**; compression names PCM, AAC-LC, AAC-ELD, H.264, Opus.
  Opus encode and decode (`_AudioConverterFillComplexBufferOpusEncode/Decode`, `airplay::COPUSDecoder`),
  backed by `eso/lib/libopus.so`. **[P-413]**
- 210.81: PCM, `ALAC/44100|48000/16|24/2`, `AAC-LC/44100|48000/2`, `AAC-ELD/44100|48000/2`. No Opus, no
  mono AAC-ELD.
- `dio_manager.json` (MH2p `efs-system/etc/eso/production/dio_manager.json`) comment: "For CarPlay over
  USB, the ANN_M PCM: @44.1kHz/16bits/mono (1764 bytes) / For CarPlay over WiFi, the ANN_M OPUS:
  @48kHz/16bits/mono (1920 bytes)". `downlinkFragSizeInMs` has separate `mainAudioWireless`
  (music 10, speech 10, telephony 16) and `altAudioWireless` 10, same values as USB. Latencies
  (`default` in 95 / out 80, `telephony` and `speech` 85/80) are not split by transport. 320.17.1 also
  carries `AUDIO_DOWNLINKFRAGSIZEINMS_*WIRELESS*` key names (0xf1704..). **[P-414]**

### 2.9 Timing, keep-alive, `/info`

- Timing: 320.17.1 has `_TimingInitialize/_TimingNegotiate/_TimingSendRequest/_TimingThread`,
  `"Timing set up on port %d to port %d"` (`FUN_00041e30`), `timingPort`, `AirTunesClock_*`,
  `UpTicksToNTP`, `NTPtoUpTicks`. No `PTP`, `timingProtocol` strings in either library. 210.81 has
  `NTPClock*` and `AirPlayNTPClient_*`. Both use NTP-style timing; no evidence of PTP. **[P-415]**
- Keep-alive: 320.17.1 has `AirPlayKeepAliveReceiver`, `keepAlivePort`, `keepAliveLowPower` (2),
  `keepAliveSendStatsAsBody`, screen opcode `kAirPlayScreenOpCode_KeepAliveWithBody`. 210.81 has
  `keepAliveSendStatsAsBody` but not `keepAliveLowPower`. **[P-416]**
- `/info` and setup keys in 320.17.1 (string block 0xc2000..0xc9000): `audioLatencies`, `bluetoothIDs`,
  `displays`, `extendedFeatures`, `hardwareRevision`, `hidDevices`, `hidLanguages`,
  `keepAliveSendStatsAsBody`, `keepAliveLowPower`, `limitedUIElements`, `limitedUI`, `manufacturer`,
  `modes`, `name`, `nightMode`, `oemIcon(s)`, `oemIconLabel`, `oemIconVisible`, `OSInfo`, `rightHandDrive`,
  `txtAirPlay`, `vehicleInformation`, `deviceID`, `interfaceName`, `macAddress`, `transportType`,
  `timingPort`, `eventPort`, `keepAlivePort`, `redundantAudio`, `usingScreen`, `timelineOffset`,
  `osBuildVersion`, `sessionUUID`. New against 210.81: `extendedFeatures`, `keepAliveLowPower`,
  `oemIcons`, `vehicleInformation`, `redundantAudio`, `usingScreen`, `keepAlivePort`, `transportType`,
  command `updateVehicleInformation`. **[P-417]**
- Transport-type names `Enet`, `WiFi`, `Direct`, `BTLE` exist in both libraries.

### 2.10 Video

320.17.1 decodes with NVIDIA NvMedia (`gLogCategory_NvMediaVideoDecoder*`, NEEDED `libnvmedia.so`,
`libnvparser.so`) and renders with `QNXScreenRenderDM`; 210.81 uses Qualcomm OMX
(`gLogCategory_OMXVideoDecoder`, `libOmx*`). **[P-418]**

---

## 3. dio_manager: how the wireless session is built (MH2p)

`app/eso/bin/apps/dio_manager` (958,480 bytes, `ET_DYN`, BIND_NOW). All functions below are Ghidra
addresses in that file.

### 3.1 AirPlay thread (`FUN_0008f8f8`, 1,144 bytes; log tag `threadLoop`) **[P-420]**

1. `videoDumping = getValueBool(0x52 SCREEN_VIDEODUMPING)`.
2. `AirPlayReceiverServerCreate(this+0xc)`.
3. **Interface selection** (config key IDs from `dio::CDioManagerComp::toString` at 0x68e10: 0x59
   `WIFI_FORCE`, 0x5a `WIFI_IFNAME`, 0x5b `WIFI_IPADDR`):
   ```text
   if (!WIFI_FORCE && !isWifiConnection(this->ctx+0x328))    ifname = "carplay0"
   else if (getIFName_ipv4(WIFI_IPADDR, &name))              ifname = name   // log "<DIO> wifi interface: %s (%s)"
   else if (file "/tmp/brcm_wifi" exists)                    ifname = "bcm0"
   else                                                      ifname = "uap0"
   ```
   `getIFName_ipv4` (`FUN_000a8038`, symbol `dio::getIFName_ipv4`) walks `getifaddrs()`, takes `AF_INET`
   entries, formats each with `getnameinfo(..., NI_NUMERICHOST)` and returns the `ifa_name` whose address
   `strcasecmp`-equals the configured IP. With the shipped config (`wifi.ipaddr "10.174.189.1"`) this is
   the 5 GHz AP `bcm0` (P-109, P-201). `WIFI_IFNAME` is **not** read in this function; this refines
   P-217, which assumed the configured ifname is used first.
4. `CFObjectSetPropertyCString(server, 0, AirPlayReceiverServerSetProperty, 1, "interfaceName", ifname)`
   (property constant at 0xd3fe0 → `interfaceName`), log `"<interfaceName>: %s"`; then optionally `model`
   (log `"<MODEL> Name"`).
5. `AirPlayReceiverServerSetDelegate`, `AirPlayReceiverServerSetWindowsOffset`,
   `AirPlayReceiverServerSetMaxFps(getValueInt(0x4d SCREEN_MAXFPS))` (config `screen.maxFPS 60`).
6. `CarPlayControlClientCreateWithServer(&client, server, s_carPlayControlClientEventCallback, this)`.
7. `AirPlayReceiverServerControl(server, "startServer")` (constant 0xd3ff8), `CarPlayControlClientStart`,
   `CFRunLoopRun()`; on exit `CarPlayControlClientStop` and the stop control.

The CarPlay control client is created and started for **every** session, wired or wireless; with
`carplay0` it will simply find no `_carplay-ctrl` controller on that interface (inference).

### 3.2 Connection type and "transport == 2" **[P-421]**

- `FUN_00072974` maps the SI connection type: 0 `UNKNOWN`, 1 `USB`, 2 `WIFI`. `FUN_00072df4` logs the
  whole record: `"connectionType=%s,localBtMacAddress=0x%llx,usbInfo=%s,wifiInfo=%s"` with
  `wifiInfo = [ipAddress, macAddress, port, transportDeviceName]`.
- `FUN_00096ee8(ctx+0x328)` returns true when the optional connection record is set and its type field is
  `2`. So "transport == 2" in the interface selection is **Wi-Fi** (proven for this enum).
- Separately, `s_sessionCreatedCallback` (0x911b8) logs the AirPlay session's NetTransportType from
  `session+0x80`: 1 `Enet`, 2 `WiFi`, 8 `USB`, 0x10 `Direct`, 0x20 `BTLE`, 0x40 `WFD`. There is no
  `SetTransportType` call from `dio_manager`; the library determines it.

### 3.3 Accepting the phone's `_carplay-ctrl` service **[P-422]**

`s_carPlayControlClientEventCallback` (0x8f0d8, 1,692 bytes):

| Event | Behaviour |
| --- | --- |
| 0 `Add/Update` | If no controller is selected yet: `CarPlayControllerGetInterfaceName`; log `"Discovered device on interface: "%s" \| carplay interface: "%s""`. Only if it `stricmp`-equals the `interfaceName` chosen in §3.1: `CarPlayControllerCopyName`, `CarPlayControllerGetBluetoothMacAddress`. If the stored target BT MAC (`this+0x48/0x4c`) is zero it adopts the discovered one. If the discovered MAC equals the target, it calls `CarPlayControlClientConnect(client, controller)` up to 5 more times with a sleep between tries (`"Error: %d \| retries remained=%d"`, final `"HTTP connection failed"`), then stores the controller (`"Device ("%s") connected!"`) and posts the BT MAC to the controller object (+0x78/+0x7c). |
| 1 `Remove` | Clears the selection if it is the selected controller. |
| 2 `Stopped` | Logged. |

This is the phone ↔ session association on MH2p (MHI2 STATUS Q7): the `_carplay-ctrl._tcp` record's
device ID is compared, as a Bluetooth MAC, with the MAC of the phone the session was started for.
`smartphone_integrator` browses the same service first (P-104, P-313) and passes `wifiInfo` to
`dio_manager`.

### 3.4 AirPlay deviceID **[P-423]**

- MH2p: `serverCopyPropertyCallback` (`FUN_0008c22c`) answers the device-ID property through
  `getDeviceID` (`FUN_0008c0b4`), called with interface `NULL`. `FUN_000a6d14(ifname)` would return an
  interface's `AF_LINK` MAC via `getifaddrs`, but returns 0 for `NULL`; the code then parses a stored MAC
  string (`FUN_00070d24`), flips bit 0 of the 48-bit value (`^ 1`) and formats it. Log:
  `"<INFO> deviceID: %s (BT MAC: %s) | interface: %s"`. **Partial**: the stored string is logged as the
  BT MAC; its source field was not traced.
- MU1329: `FUN_0015c204` (the equivalent AirPlay setup in MU1329 `dio_manager`) calls
  `FUN_00175ad0("uap0", buf)` to get the `uap0` MAC, logs `"<devicedID>: %s (interface: %s)"`, sets
  `deviceID`, then sets **`interfaceName` = `"carplay0"`** unconditionally and `model` =
  `"AirPlayGeneric1,1"` or the configured name (constants resolved from the GOT at 0x1a5dc0: 0x197810
  `uap0`, 0x195ee8 `interfaceName`, 0x19784c `carplay0`, 0x197880 `AirPlayGeneric1,1`). This confirms
  07-mhi2q-stock-carplay.md ("deviceID from the uap0 MAC").

### 3.5 iAP2-side wireless functions in dio_manager (strings) **[P-424]**

Consistent with P-312: `CIAP2ServiceCinemo` logs `"<iAP2> CarPlay over WiFi, available: %d"`
(`WirelessCarPlayUpdate`), `setWiFiTransportComponent` / `"WiFi component: transportName = %s"`
(`FUN_000aee08`, Cinemo `CinemoWiFiComponent`), OOB Bluetooth pairing (`StartOOBBTPairing`,
`OOBBTPairingAccessoryInformation`, `…CompletionInformation`, `…LinkKeyInformation`; log
`"<OOB_BT_PAIRING> linkKey: %s, mac: %s"`), `CIAP2WiFiInformationSharing`
(`requestAccessoryWiFiConfigurationInformation` → `wiFiInformation`), and
`deviceTransportIdentifierNotification` (`bluetoothTransportIdentifier`, `usbTransportIdentifier`)
forwarded to SI via `ISmartPhoneAppProxyReply::iap2DeviceTransportIdentifierNotification`. Technology
strings `WIRELESS_ONLY`, `USB_AND_WIRELESS` (log `"<iAP2> BT MAC address (%llu): %s | Technologies: %s"`,
`FUN_0008ff4c`). `dio_manager.json` `bt.connectionTimeoutMs 2000` is commented "Waiting on BT stack for
obtaining the MAC address, required for having OOB BT pairing"; `bt.fakeMacAddr "AA:BB:CC:DD:EE:FF"`.

MU1329 `dio_manager`: 0 hits for `OOBBTPairing`, `WiFiInformation`, `wirelessCarPlay`,
`CarPlayControl`, `CinemoWiFiComponent`, `getIFName`, `WIFI_IFNAME`, `brcm_wifi`.

### 3.6 Coding gate **[P-425]**

`DIOCodingProvider` (`FUN_000a6624`, handler `onEvent_updateBlob`) dispatches persistence updates by
(partition, key). Constants read from the data words it compares against: (28180695, 1) → `onCodingChanged`
(brand, carClass, carGeneration, carDerivate, carDerivateSupplement); (…, 1322) → `onLimitationChanged`
(nhtsaProperties); (46924065, 401) → `onIdentificationMuTrainChanged`; key **8877** →
`onVehicleConfiguration`:

```text
usb  = blob.size > 8    && (blob[8]    & 0x01)      // FUN_000711b8
wifi = blob.size > 0x17 && (blob[0x17] & 0x08)      // FUN_000711fc
enCarPlayTechnology = usb | (wifi ? 2 : 0)          // log "enCarPlayTechnology changed from %s to %s"
```

Enum names: `CARPLAYTECHNOLOGY_UNKNOWN`, `_USB_ONLY`, `_WIFI_ONLY`, `_BOTH` (so 1 = USB, 2 = Wi-Fi,
3 = both; 0 = unknown is inferred from the order). **Partial**: which coding/adaptation channel feeds key
8877 is not resolved. SI separately checks `carplayWireless` (`"[gal=%d|galWireless=%d|carplay=%d|
carplayWireless=%d|…]"`) and the 5 GHz AP (P-105, P-214).

MU1329 `dio_manager` has no `CARPLAYTECHNOLOGY` / `enCarPlayTechnology` strings.

---

## 4. HMI (lsd.jxe) and coding strings

Byte search of `stage2_ifs5/ifs/lsd.jxe` (138 MB) and MU1329 `<MU1329>/ifs/lsd.jxe`; no
decompilation. **[P-426]**

| String | MH2p | MU1329 |
| --- | --- | --- |
| `de/audi/app/terminalmode/hmi/WirelessCarplayHMIModelHandler` (+ `$1..$3`) | 10 | 0 |
| `de/audi/app/terminalmode/interapp/WlanWaker`, `BluetoothWaker` | 11 / present | 0 |
| `SERVICETYPE_CARPLAY_OVER_WIRELESS`, `CARPLAY_OVER_WIRELESS` | 2 | 0 |
| `CARPLAY_WIRELESS` | 1 | 0 |
| `getSmartphoneTypeWhichSupportsWireless` | 1 | 0 |
| `ATTR_GETTECHNOLOGYSELECTION` (`DSIPhoneManager`), `requestRejectTechnologySelection` | 2 / 3 | 0 |
| `CONNECTIONTYPE_WLAN` | 2 | 1 |
| `requestBTDeactivation` | 5 | 3 |

MH2p's HMI has a dedicated wireless CarPlay model handler that can wake WLAN and Bluetooth, and a
technology-selection DSI; MU1329's HMI has only the `CONNECTIONTYPE_WLAN` constant (unused, per
08-mhi2q-wireless-plan.md). No `smartphone-versions` or feature-flag string for wireless CarPlay was found in
either `lsd.jxe` beyond these.

---

## 5. Answers for the MHI2 repo

| MHI2 item | What MH2p shows |
| --- | --- |
| **STATUS Q5** — where `interfaceName` is populated | By `dio_manager`, immediately after `AirPlayReceiverServerCreate`, through `CFObjectSetPropertyCString(server, …, AirPlayReceiverServerSetProperty, …, "interfaceName", name)`; `libairplay` copies it into its object (320.17.1: obj+0x1a1; 210.81: obj+0x16c). MH2p value: `carplay0` wired, the AP interface owning 10.174.189.1 wireless (P-409, P-420). MU1329 does the same with a constant `carplay0` (P-423). Inference for MU0678: look for the same `CFObjectSetPropertyCString` + `interfaceName` constant in MU0678 `dio_manager`; the obj+0x6c field (E-026) is that property's storage. |
| **STATUS Q6** — callers/arguments of packet/multicast helpers | Not traced. MH2p still exports `SocketSetMulticastInterface` (276 bytes), `SocketSetPacketReceiveInterface` (48), `SocketSetBoundInterface` (10, stub-sized, as E-019). The production wireless path does not need them for Bonjour: the interface binding that matters is `if_nametoindex(interfaceName)` → `DNSServiceRegister`, with `mdnsd` running globally (P-111). |
| **STATUS Q7** — how BT/iAP2 and Wi-Fi/AirPlay are tied to one phone | Bluetooth MAC. SI starts `dio_manager` with `localBtMacAddress` + `wifiInfo` (P-313, P-421); `dio_manager` accepts only the `_carplay-ctrl._tcp` controller on its chosen interface whose device ID equals the target BT MAC (P-422); iAP2 `DeviceTransportIdentifierNotification` merges USB/BT identities (P-311, P-424). |
| **STATUS Q8** — can the AirPlay/mDNS path run on `uap0` unmodified | On MH2p the *same* code path serves `carplay0` and a Wi-Fi AP interface, and still names `uap0` as its fallback (P-420). The mechanism is interface-agnostic: `interfaceName` + a global `mdnsd`. For MU0678/MU1329 the library side needs only a different `interfaceName` value; what is missing is the wireless trigger (`_carplay-ctrl` browse + `/ctrl-int/1/connect`), HomeKit pairing and the iAP2-over-AirPlay command handling, none of which exist in 210.81 (P-403, P-407, P-411). That last part is inference for MU0678 (its library was not examined here). |
| E-018 / E-019 (setter stubs) | Same stubs in MU1329 210.81 (P-402); removed in 320.17.1. |
| E-026 (`interfaceName` → `if_nametoindex` → `DNSServiceRegister`) | Same in 210.81 and 320.17.1 (P-404). |
| TRACE-004 / TRACE-005 | The DIO → AirPlay server sequence of §3.1 is the MH2p equivalent; TRACE-005's open socket-helper question stays open. |

---

## 6. What this means for MHI2 / MHI2Q wireless CarPlay

Facts:

- A production VW/Audi receiver runs wireless CarPlay with AirPlay 320.17.1, which adds, relative to
  MU1329's 210.81: HomeKit pair-setup/pair-verify, the `_carplay-ctrl` browse + `/ctrl-int/1/connect`
  trigger, `iAPSendMessage` iAP2 tunnelling, Opus audio and low-power keep-alive.
- `dio_manager` on MH2p adds the Wi-Fi interface selection, the BT-MAC match, `disableBluetooth` BT
  release, Cinemo Wi-Fi/BT transport components, OOB BT pairing and Wi-Fi information sharing. MU1329's
  `dio_manager` has none of the wireless parts but already has the `interfaceName` property path.

Inference (not proven for MHI2/MHI2Q):

- Changing only `interfaceName` (or `MDNS_DIRECTLINK_IFACE`) on MU1329/MU0678 would at most make the
  receiver advertise on the AP. The phone would still need a Wi-Fi-capable receiver: pairing, the connect
  trigger and an iAP2 path over the session. 210.81 lacks all three, so a wireless MHI2Q receiver needs
  either a different AirPlay implementation (e.g. an xcertplay/LIVI-style one) or the 320.17.1 library
  running on MU1329 (see [portability.md](portability.md) for why that is not a drop-in).
- The MH2p sequence is a usable reference design: SI browses `_carplay-ctrl._tcp` on the AP interface;
  the session owner registers `_airplay._tcp` bound by interface index; it sends `GET /ctrl-int/1/connect`
  with `AirPlay-Receiver-Device-ID` to the controller whose device ID matches the BT MAC seen over iAP2;
  iAP2 then continues inside the session via `iAPSendMessage` in both directions; `disableBluetooth`
  releases the RFCOMM link.
- MH2p's deviceID is derived from a MAC (BT-MAC-like value with bit 0 flipped), MU1329's from the `uap0`
  MAC. Either form is a stable 6-byte ID; what matters for the trigger is that
  `AirPlay-Receiver-Device-ID` equals the TXT `deviceid` (inference from the request construction).

---

## 7. Open / not determined

- The source field of the MAC used by MH2p `getDeviceID` (logged as "BT MAC"), and the exact bit meaning of
  the `^ 1` (P-423).
- Which coding / adaptation value feeds persistence key 8877 (P-425), and the partition for that key.
- Whether a 320.17.1 receiver ever receives stream 130 from current iOS (the library would reject it as
  unsupported; P-410, P-412).
- The full `/info` reply layout of 320.17.1 (keys listed from strings, not from `_requestProcessInfo`
  decompilation).
- Callers and arguments of `SocketSetMulticastInterface` / `SocketSetPacketReceiveInterface` (STATUS Q6).
- An exhaustive list of `DNSServiceRegister` call sites in 320.17.1 (P-405).
- MU0678's `libairplay` version and whether it has any of the 320.17.1 features (no MU0678 binary was
  available for this analysis).

---

## Evidence

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
