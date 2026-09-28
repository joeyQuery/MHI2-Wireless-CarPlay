# Portability of MH2p Wireless CarPlay Binaries to MHI2Q (MU1329) and MHI2 (MU0678)

Can the MH2p binaries that implement wireless CarPlay run on an MHI2Q MU1329 unit (or an MHI2 MU0678
unit), and what can be reused instead? Companion to [airplay-dio.md](airplay-dio.md).

> **Scope rule.** Everything here is static analysis (ELF headers, attributes, `DT_NEEDED`, dynamic
> symbol tables). Nothing was run on a unit or in an emulator. Symbol resolution proves that a *name*
> exists, not that the ABI behind it matches. MU0678 statements are inference: no MU0678 binary was
> available for this analysis. Evidence IDs P-470..P-499 are listed at the end.

Paths: MH2p relative to `<MH2p>/`; MU1329 relative to `<MU1329>/` (see [README](README.md#paths))
unless stated. MU1329 boot-IFS libraries (not in the earlier extracts) were extracted read-only from
the MU1329 update package's `MMX2QC/eifs/10/default/eifs.mbn` and `mifs-stage2/10/default/ifs2.img` (nested
`ifs_coreservices3/ifs_display/ifs_startup`) for this comparison. The MH2p `libc.so.3`
is not in any extracted MH2p image (it lives in the stage-1 IFS), so MH2p imports were checked against
MU1329's libc directly.

---

## 1. Summary verdicts

| MH2p binary | Verdict on MU1329 | Main blockers | Reusable as reference |
| --- | --- | --- | --- |
| `eso/lib/libairplay.so` (320.17.1) | **Not portable** | NVIDIA NvMedia/parser video decode (Tegra K1 driver stack), `libcpp-ne.so.5`, MH2p `libdisplayinit` ABI, eso/Cinemo symbols from the host process | Protocol behaviour: HomeKit pairing, `_carplay-ctrl` trigger, TXT keys, `iAPSendMessage`, Opus formats, stream table |
| `eso/bin/apps/dio_manager` | **Not portable** | Needs 320.17.1 `libairplay`, `libesoiap2`, `libcpp-ne.so.5`, newer eso framework ABI (`libiplcommon`, `libcomm`, `libutil`, `libdsicommon`), NVIDIA libs, `quick_exit`; PIE executable; serves a newer `DSICarplay` to the HMI | Session sequence, interface selection, BT-MAC match, config keys and values |
| `eso/bin/apps/iap2connectionmanager` | **Not portable** | `libesoiap2`, `libcpp-ne.so.5`, eso framework ABI; PIE | iAP2 link parameters for BT vs USB (P-308), Wi-Fi config sharing flow |
| `eso/lib/libesoiap2.so` | **Needs shim, impractical** | `libcpp-ne.so.5` and 11 newer `libiplcommon` symbols | Message coverage of an ESO iAP2 client with wireless modules |
| `armle/usr/lib/cinemo/libEsoIAPTransport.so` | **Not portable** | eso framework (`libcomm`, `libosal`, `libutil`, `libiplcommon`) + `libcpp-ne.so.5` | Transport plug-in interface for `iap://bt://` (P-317) |
| Cinemo NME set (`libNmeBaseClasses`, `libNme`, `libNmeImage`, `libNmeSDK`, `cinemo/libNmeVfs`, `libNmeTransport`, `libNmeAppleAuth*`) | **Candidate as a set, needs shim + testing** | Must be replaced as a whole; needs `libpng14.so.0`; 1 SDK symbol missing for MU1329 `dio_manager`; plug-in ABI, licensing and behaviour unknown | Wireless iAP2 identification (BT + WirelessCarPlay components), OOB BT pairing, Wi-Fi info sharing |
| `eso/lib/libopus.so` | **Drop-in (library level)** | Only a trivial `__aeabi_idiv0` import besides libc/libm | Opus codec for wireless audio |
| `libcpp-ne.so.5` (`stage2_ifs1/lib`) | Loadable in isolation | Only `__aeabi_ldiv0` missing | — but mixing it with MU1329's `libcpp-ne.so.4`/`libecpp-ne.so.4` users in one process is not viable |

---

## 2. Platform facts

### 2.1 SoC and OS **[P-470]**

| | MH2p | MHI2Q MU1329 | MHI2 MU0678 (per MHI2 repo) |
| --- | --- | --- | --- |
| SoC | NVIDIA Tegra K1 (`nv_tk1`; `stage2_ifs3/lib/libcapture-soc-t124.so`) | **Qualcomm APQ8064** (see below) | NVIDIA Tegra 3 (`devu-iap2-tegra3-ci.so`, MHI2 `docs/firmware.md`) |
| OS | QNX 6.6.0 | QNX 6.5.0 SP1 (`libc.so.3` build path `PSP_kernel-libc-nvidia_brCustom_be650SP1`) | QNX 6.5 family (not verified here) |
| Compiler of CarPlay binaries | GCC 4.9.2 | GCC 4.4.2 | not examined |
| C++ runtime | `libcpp-ne.so.5` (Dinkumware) | `libcpp-ne.so.4`, `libecpp-ne.so.4` | not examined |
| Video decode in AirPlay | NvMedia (`libnvmedia`, `libnvparser`) | Qualcomm OMX (`libOmxCore`, `libOmxBase`, `OMX.qcom.video.decoder.avc`) | not examined |

**Correction to the shared brief:** MU1329 (`MMX2QC`) is Qualcomm, not Tegra 3. Evidence:
`firmware/MMX2QC/qcbl/` (`qc-bootloader.bin`), `.mbn` image containers (`mifs.mbn`, `eifs.mbn`), and boot-IFS
libraries `libhwio_asic_8064.so`, `libpowermgr_asic_8064.so`, `libpmic_npa-mmx2-8064.so`,
`libgpio_intr_asic_8064.so`, `libadreno_utils.so`, `libGSLUser.so`, `libcsd_oem_lib-mmx2_8064.so`; the
CarPlay decoder is `OMX.qcom.video.decoder.avc` (separate MU1329 analysis). APQ8064's Krait cores implement
ARMv7-A with VFPv4-D32 and NEON (general ARM knowledge, not from the firmware).

### 2.2 Instruction set and float ABI **[P-471]**

`readelf -h -A` (container `qnx65-armv7-toolchain:8.5`):

| File | `e_flags` | `Tag_FP_arch` | `Tag_Advanced_SIMD_arch` | Mode |
| --- | --- | --- | --- | --- |
| MH2p `libairplay.so`, `dio_manager`, `iap2connectionmanager`, `libesoiap2.so`, `libEsoIAPTransport.so`, `libopus.so`, `smartphone_integrator`, `libNmeVideoNvMedia.so` | 0x5000202 (EABI5, `EF_ARM_ABI_FLOAT_SOFT`) | VFPv3 (32 D-regs) | NEONv1 | Thumb-2 |
| MU1329 `libairplay.so` | 0x5000002 | VFPv3-D16 | — | ARM |
| MU1329 `dio_manager` | 0x5000002 | VFPv3 | — | ARM |

- Both sides use the soft-float calling convention (`Tag_ABI_HardFP_use: Deprecated`, no VFP-args tag);
  the 0x200 flag only states it explicitly. No calling-convention conflict.
- VFPv3-D32 + NEON is **not** a blocker on MU1329: a scan of all ELF files in the MU1329 extract and boot
  IFS counted 19 ELF files built for VFPv3 + NEONv1 (e.g. `armle/usr/lib/libNmeSDK.so`; a few
  libraries appear twice in that count because they were copied for the comparison) and 7 for
  VFPv3-D16 + NEON (e.g. `cinemo/libNmeAudioDolby.so`). The unit already runs NEON code. For MU0678 (Tegra 3, Cortex-A9 with NEON) the same
  is expected (inference).

### 2.3 Executable type, binding and symbol versions **[P-472]**

- MH2p executables (`dio_manager`, `iap2connectionmanager`, `smartphone_integrator`) are `ET_DYN`
  (position-independent executables) with interpreter `/usr/lib/ldqnx.so.2`; MU1329 `dio_manager` is
  `ET_EXEC`. Whether QNX 6.5.0 SP1's loader starts a PIE correctly is **not verified** (QNX introduced
  PIE/ASLR support officially with 6.6; treat as unknown). It can be checked in the repo's QNX 6.5 QEMU
  harness with a trivial PIE.
- MH2p `dio_manager` is linked `BIND_NOW` (`FLAGS_1: NOW`): every undefined symbol must resolve at load
  time, so a single missing symbol stops the process.
- The only symbol-version requirement is `libsocket.so.3` version `libsocket.so.2`. MU1329's
  `libsocket.so.3` (eifs) defines both `libsocket.so.3` and `libsocket.so.2` version definitions, as
  does MH2p's. Not a blocker.

---

## 3. Dependency check per binary

Method: for each MH2p binary, every strong undefined dynamic symbol was looked up in the MU1329 library
of the same `DT_NEEDED` name (plus MU1329 `libc.so.3` and `libm.so.2`); unresolved ones were attributed to
the MH2p library that defines them. Weak CRT symbols (`_ITM_*`, `_Jv_RegisterClasses`,
`__register_frame_info`) are ignored. **[P-473]**

### 3.1 `DT_NEEDED` availability on MU1329

| NEEDED | On MU1329? | Where |
| --- | --- | --- |
| `libc.so.3`, `libm.so.2`, `libsocket.so.3` | yes | `mifs`/`eifs` `proc/boot` |
| `libscreen.so.1`, `libasound.so.2`, `libbacktrace.so.1`, `libusbdi.so.2`, `libpps.so.1` | yes | `eifs` / `ifs2` nested images |
| `libcomm.so`, `libosal.so`, `libutil.so`, `libiplcommon.so`, `libdsicommon.so`, `libservmngt.so` | yes, **older ABI** | `ifs_coreservices3` `mnt/app/eso/lib` |
| `libdns_sd.so.1`, `libnbutil.so.1`, `libNmeSDK.so`, `libair.so`, `libdisplayinit.so` | yes, older versions | `app/armle/usr/lib`, `app/eso/lib` |
| `libcpp-ne.so.5` | **no** (only `libcpp-ne.so.4`, `libecpp-ne.so.4`, `libcpp.so.4`) | — |
| `libopus.so` | **no** | — |
| `libnvmedia.so`, `libnvparser.so` (and their `libnvrm.so`, `libnvtvmr.so`, `NvOs*`) | **no** (Qualcomm unit) | — |
| `libesoiap2.so`, `libairplay.so` 320.17.1 | **no** / different version | — |
| `libpng14.so.0` (Cinemo SDK) | **no** (only `armle/webkit/libpng12.so.0`) | — |

### 3.2 Unresolved symbols against MU1329 **[P-474]**

| MH2p binary | Strong undefined | Unresolved on MU1329 | Breakdown |
| --- | --- | --- | --- |
| `libairplay.so` | 324 | 97 | `libcpp-ne.so.5` 40; NvMedia 15; Opus 12; `libnvparser` 5 (`video_parser_*`); `libdisplayinit` 5 (`dint_*`, MU1329 version lacks them); 20 from the host process (`CinemoCreateAudioCodec/Config`, `osal::MutexOps`, `tracing::Channel`) |
| `dio_manager` | 507 | 215 | `libairplay` 78; `libcpp-ne.so.5` 68; `libesoiap2` 36; `libiplcommon` 22 (newer `std::ErrorStorage`, `GlobalErrorHandler`, `UUID`, `tracing::Channel` ctor); `libNmeSDK` 3; `libcomm` 3; `libutil` 2; `libdsicommon` 1; `libair` 1; libc `quick_exit` 1 |
| `iap2connectionmanager` | 205 | 75 | `libcpp-ne.so.5` 29; `libesoiap2` 26; `libiplcommon` 18; `libcomm` 1; `__aeabi_idiv0` 1 |
| `libEsoIAPTransport.so` | 115 | 44 | `libcpp-ne.so.5` 35; `libiplcommon` 8; `libcomm` 1 |
| `libesoiap2.so` | 84 | 32 | `libcpp-ne.so.5` 20; `libiplcommon` 11; `__aeabi_idiv0` 1 |
| `smartphone_integrator` | 354 | 68 | `libcpp-ne.so.5` 38; `libiplcommon` 22; `libosal` 2; `libcomm` 2; others 4 |
| `libopus.so` | 21 | 1 | `__aeabi_idiv0` |
| `libcpp-ne.so.5` | 118 | 1 | `__aeabi_ldiv0` |
| `libdisplayinit.so` (MH2p) | 36 | 0 | — |
| `libnvparser.so` | 23 | 5 | `libnvrm` 3, `NvOs*` 2 |
| `libnvmedia.so` (`stage2_ifs3/lib`) | 177 | 118 | `libnvtvmr` 96, `libnvrm` 14, `NvOs*` 8 |
| Cinemo `libNmeBaseClasses.so` | 244 | 0 | — |
| Cinemo `libNme.so` | 271 | 20 | 19 from `libNmeBaseClasses` (satisfied by the MH2p version), `__aeabi_ldiv0` |
| Cinemo `libNmeSDK.so` | 2,023 | 265 | 263 from `libNmeBaseClasses`, 1 `libNme`, 1 `libNmeImage` (all MH2p set) |
| Cinemo `cinemo/libNmeVfs.so` | 1,378 | 143 | all from `libNmeBaseClasses` (MH2p set) |
| Cinemo `libNmeTransport.so`, `libNmeAppleAuthImpl.so` | 95 / 95 | 5 / 13 | `libNmeBaseClasses` (MH2p set), `__aeabi_*div0` |

### 3.3 libc gap (QNX 6.6 → 6.5) **[P-475]**

Across all MH2p binaries above, the C-level imports missing from MU1329's `libc.so.3` / `libm.so.2` /
`libsocket.so.3` are only:

- `quick_exit` (C11; `dio_manager`),
- `__aeabi_l2f` (`libairplay.so`), `__aeabi_idiv0` / `__aeabi_ldiv0` (several): libgcc helpers a shim
  library can provide trivially.

All other libc/libm/libsocket names resolve. This is a name-level result: QNX 6.6 headers may change
structure layouts or kernel-call semantics that name resolution cannot see (inference).

### 3.4 C++ runtime and framework ABI **[P-476]**

- `libcpp-ne.so.5` is missing on MU1329 but is itself loadable there (only `__aeabi_ldiv0` missing).
- The blocker is the **ESO framework**: MU1329's `libiplcommon`, `libcomm`, `libosal`, `libutil`,
  `libdsicommon`, `libservmngt` are GCC 4.4.2 builds against `libcpp-ne.so.4`/`libecpp-ne.so.4` and lack
  22 (`dio_manager`) / 18 / 11 / 8 symbols the MH2p binaries need (e.g.
  `_ZN7tracing7ChannelC1ERKSssj`, `std::ErrorStorage::*`, `std::GlobalErrorHandler::*`, `std::UUID::*`).
  Shipping the MH2p versions of these libraries would put two ESO frameworks and two C++ runtimes
  (`.4` and `.5`, same `std::` names) into the address space of anything that also loads MU1329 libraries,
  and the framework libraries carry the IPC/DSI protocol to the MU1329 HMI and services. Not viable as a
  drop-in (inference from the symbol sets; not tested).
- MH2p `dio_manager` serves `DSICarplay` (generated from `.../dsi/carplay/DSICarplayS.hxx`) and a newer
  `ISmartPhoneAppProxy` to MH2p's SI and HMI; MU1329's HMI speaks an older DSICarplay (separate MU1329 analysis;
  08). Interface-version compatibility was not checked; mismatches are likely (inference).

### 3.5 Video: NVIDIA vs Qualcomm **[P-477]**

- 320.17.1 `libairplay.so` calls `NvMediaDeviceCreate`, `NvMediaVideoDecoderCreate/Render/…`,
  `NvMediaVideoMixer*`, `NvMediaVideoSurface*`, `NvxScreenCreateNvMediaVideoSurfaceSibling`,
  `video_parser_*` directly; the decoder is compiled into the library.
- `libnvmedia.so` needs `libnvtvmr.so` (96 symbols) and `libnvrm.so` (14) plus `NvOs*`: the Tegra K1 driver
  stack. None of these exist on a Qualcomm APQ8064 unit, and a TK1 NvMedia build is unlikely to match a
  Tegra 3 driver stack on MU0678 either (inference; MU0678 not examined).
- Making 320.17.1 decode video on MU1329 would need a replacement for ~20 NvMedia/parser entry points
  backed by Qualcomm OMX and QNX Screen. That is a re-implementation of the decode path, not a shim.

### 3.6 Cinemo NME: the one set that is internally consistent **[P-478]**

- The MH2p Cinemo libraries depend only on QNX system libraries (`libc`, `libm`, `libsocket`, `libssl`,
  `libcrypto`, `libusbdi`) and on each other; `libNmeBaseClasses.so` resolves fully against MU1329, and
  every other unresolved symbol is supplied by the MH2p Cinemo set itself. They do not use
  `libcpp-ne.so.5`. Extra dependency: `libpng14.so.0` (for `libNmeSDK`), not on MU1329.
- MU1329 `dio_manager` imports 50 symbols from `libNmeSDK.so`; the MH2p `libNmeSDK.so` exports 49 of them.
  Missing: `CINEMO_OPTION_IAP_AUTHENTICATION` (a data symbol; with `BIND_NOW`-style strict loading this
  alone would block the load unless supplied by a shim).
- MU1329 exports that the MH2p set drops: 205 in `libNmeBaseClasses`, 7 in `libNme`, 1 in `libNmeImage`
  (would affect any other MU1329 Cinemo user if the libraries were replaced system-wide).
- Both `dio_manager.json` files select the plug-in directory with `iap2.cinemoLib`
  (`/armle/usr/lib/cinemo`), and both Cinemo versions load iAP transports via `IAP_TRANSPORT_LIBRARIES`
  (P-319, P-320), so a private Cinemo directory for one process is at least configurable (inference).
- Functionally this set carries what MU1329's lacks: `WirelessCarPlayTransportComponent`,
  `AccessoryWiFiConfigurationInformation`, OOB BT pairing, `WirelessCarPlayUpdate` (in
  `cinemo/libNmeVfs.so` and `libNmeSDK.so`; 0 occurrences in MU1329's `libNmeVfs.so` and `libNmeSDK.so`;
  see P-322).
- Unknowns: Cinemo licence/key checks tied to the unit, whether MU1329's `dio_manager` drives the newer
  SDK correctly, and whether the MH2p `libNmeAppleAuth*` works with MU1329's MFi chip path.

---

## 4. Answers for the MHI2 repo

| MHI2 item | Answer from this analysis |
| --- | --- |
| Roadmap: reuse existing production code for wireless | MH2p's wireless CarPlay code cannot be dropped into MHI2Q or MHI2: the AirPlay library is tied to NVIDIA TK1 video and to a newer ESO/C++ runtime (P-474..P-477). |
| STATUS Q8 (AirPlay/mDNS on `uap0` unmodified) | The MH2p code shows the interface is a parameter (`interfaceName`), not a hard dependency (airplay-dio.md P-420). The missing pieces on MHI2-era libraries are protocol features, not interface binding. |
| Binary inventory (`docs/binaries.md`) | Only `libopus.so` is a clean library-level candidate; the Cinemo set is a conditional candidate for MHI2Q only (MU0678 has no Cinemo per the MHI2 repo's stack description). |

---

## 5. What this means: realistic options for MHI2Q wireless

Facts:

1. MU1329's stock receiver (210.81 + MU1329 `dio_manager`) has none of the wireless-specific protocol code
   (airplay-dio.md P-403, P-407, P-411, P-424).
2. MH2p's receiver has it, but its binaries are not portable: NVIDIA video, `libcpp-ne.so.5`, newer ESO
   framework ABI, newer DSI, PIE executables (P-472..P-477).
3. Library-level exceptions: `libopus.so` (drop-in) and the MH2p Cinemo set (self-consistent, needs
   `libpng14`, one SDK symbol and testing).

Options, with labelled inference:

| Option | What it takes | Assessment (inference) |
| --- | --- | --- |
| A. Run MH2p `dio_manager` + 320.17.1 on MU1329 | Replace video decode (NvMedia → OMX), ship `libcpp-ne.so.5` and the MH2p ESO framework, bridge DSI to the MU1329 HMI, verify PIE loading | Not realistic. The video path alone is a re-implementation, and the framework/DSI mix is a stability risk on a unit the owner must not brick. |
| B. Keep MU1329's wired stack, add a separate wireless receiver | An AirPlay receiver with HomeKit pairing, `_carplay-ctrl` browse + `/ctrl-int/1/connect`, iAP2 via BT then `iAPSendMessage` (or stream 130), Opus, decode through MU1329 OMX; plus the Wi-Fi AP (`uap0`), pf rules and an iAP2-over-Bluetooth path | Realistic in design terms, large in effort. MH2p supplies the reference behaviour and values (TXT keys, interface binding, BT-MAC match, audio formats, frag sizes); xcertplay/LIVI supply a sender-side view. |
| C. Upgrade only Cinemo for iAP2 wireless identification | Private Cinemo directory for a wireless-bootstrap process, `libpng14`, a shim for `CINEMO_OPTION_IAP_AUTHENTICATION` | Possible building block for the Bluetooth iAP2 bootstrap (Wi-Fi config sharing, WirelessCarPlay component). It does not provide the AirPlay side. Needs QEMU testing first and carries licence/behaviour unknowns. |
| D. Patch 210.81 in place | Add pairing, control client, iAP2 command handling to the stock library by hooking | Not realistic: those are entire subsystems (crypto, pairing state machine, HTTP client), not seams. |

For MU0678 (MHI2) the same conclusions hold more strongly: no Cinemo, a Tegra 3 video stack unrelated to
the TK1 NvMedia build, and an older ESO framework (inference from the MHI2 repo's stack description).

---

## 6. Open / not determined

- Whether QNX 6.5.0 SP1 loads `ET_DYN` executables built for 6.6 (testable in the QNX 6.5 QEMU harness).
- Whether the MH2p Cinemo set actually runs under MU1329 `dio_manager` (licence checks, SDK behaviour,
  MFi path); only symbol names were compared.
- Structure-layout and kernel-call differences between QNX 6.6 and 6.5 headers for the libc names that do
  resolve.
- MH2p's own `libc.so.3` (stage-1 IFS, not extracted), so no comparison of which libc names are new in
  6.6 beyond what the imports show.
- MU0678 binaries (not available here): library versions, C++ runtime, video stack.

---

## Evidence

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
