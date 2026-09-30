# MH2p Firmware Provenance

This is the baseline for every `P-` evidence entry in `mh2p/`. It follows the provenance checklist in
[`docs/firmware.md`](../docs/firmware.md).

> **Scope rule.** MH2p is a different head unit (MIB2 High **Plus**, `MMX2P`), not an MHI2. Facts proven
> here are **cross-platform evidence**: they show how a production VW Group / Alpine unit with Wireless
> CarPlay is built. They are not proof of MHI2 behaviour, and must not be merged into the MHI2 evidence
> register (E-xxx) without that label.

## Required provenance

| Field | Value |
| --- | --- |
| Head unit / platform | Audi MIB2 High Plus ("MH2p"), main unit package `MMX2P` |
| SWDL release / train | `MH2p_ER_AUG33_P2873` (`MUVersion` 2873), `Meta/main.mnf` |
| Supported variants | `M2P-HS-*-EU-AU-MLE-AL`, `M2P-HS-*-RW-AU-MLE-AL` |
| Supported trains | `MH2p_ER_AUG33_?27??`, `MH2p_ER_AUG33_?28??` (and `_?` suffixes) |
| Main unit package version | `MMX2P` 2201.8.0 |
| Build identifier | `CLU28_MMX2P_AU_ER_G33_004PROD` (`img_ver.txt`), build branch `rel/CLU28_newPDK`, up to CL 45095183 |
| First-tier supplier | ALPINE (`ifs/version_info.txt`) |
| HMI / framework | `HMI=G33`; Framework `6.37.7.SR5 MIB2PMAIN I147 CI74` |
| OS | **QNX 6.6.0** (`ifs/version_info.txt`) |
| SoC / CPU / ABI | NVIDIA **Tegra K1** (`nvflash/alpinesecureboot`, PCI module `pci_hw-nv_tk1-vcm30-mib.so`), ARMv7-A, EABI5, `Tag_FP_arch: VFPv3` (D32), GCC 4.9.2, C++ runtime `libcpp-ne.so.5` |
| Wi-Fi / Bluetooth chip | Broadcom **BCM4359** (driver `devnp-qwdi-2.5_bcm4359-wapi.so`) |
| Source | Stock SWDL update package (`<MH2p SWDL>`, not distributed here) |
| Modifications | None. Stock, signed package (`main.mnf.cks`, `main.mnf_cks_S.sig`) |
| Runtime correspondence | **None.** No MH2p unit was observed. All `P-` entries are static |
| Extraction date | 2026-09-28 |

## Hashes (SHA-256)

| File | Size | SHA-256 |
| --- | --- | --- |
| `Meta/main.mnf` | 5,621 | `102b1a74878e9bb0eb3b4c6ead60ef7236801f21ab0673c23104ed17e0a10aee` |
| `Data/MMX2P.app_CLU28_MMX2P_AU_ER_G33_004PROD/20/app.img` | 2,147,418,112 | `16e22c802a782168c444a1b3ed51224b37cfcf5f5479177767cbd0b1fdcc4fca` |
| `Data/MMX2P.efs-system_CLU28_MMX2P_AU_ER_G33_004PROD/20/efs-system.img` | 2,097,152 | `620d83915ab707edc42d91f7e0dc620b792e5fa338f1fa4944512921ee5d7c15` |
| `Data/MMX2P.mifs-stage2_CLU28_MMX2P_AU_ER_G33_004PROD/20/main_stage2.img` | 32,505,856 | `304b91556cd63fca893f3adf9138c873142333b8e31029371cc46b2e28e6efa9` |
| `app.img:/img_restore/main_stage2.ifs.lzo` | 71,262,720 | `a7beb1cdea84ab1687f12f1b858dd43b3a67290a2089820d7215004f48efc85e` |
| stage2, decompressed | 176,055,808 | `28b946ccc2a773289144eae69bd7cebe335bc0275ae2bee5d079d838a7fa9580` |
| stage2 IFS 0 (fstab, `main_stage2.0.sh`) | 946,180 | `6b59177d572470e3163058ba8c7a6417e2b353e2b770d0f7c610c635e7d8da6a` |
| stage2 IFS 1 (io-pkt) | 1,901,692 | `ff8bbf2f1478457d1b072581d551c85548cdd8acbe43a292fe7e632753446085` |
| stage2 IFS 2 (NVIDIA display) | 1,942,980 | `4e456ed98869b776f178c43cbcde03116750a280a058bd434ed0a0ed28157134` |
| stage2 IFS 3 (screen, broker, framework) | 23,391,188 | `4fda3dfb0cbb7fa89f6ff72ee937d9565dbe264a5c0c42d8ac001f82d5e90bf6` |
| stage2 IFS 4 (boot scripts, drivers, Wi-Fi driver) | 5,906,716 | `d5a4a47322c3ad7727efc6b03e02d66e74843ebc191a30819f0b96fcc0b9dce4` |
| stage2 IFS 5 (`lsd.jxe`, J9) | 141,967,052 | `f0a60a53db2d698f0459702b1d6155e95d61977dc1a3877b212da87c9464ab8d` |

Hashes of the individual binaries used as evidence are in [binaries.md](binaries.md).

## Partitions and how they were read

| Image | Mounted at | Format | Reader |
| --- | --- | --- | --- |
| `app.img` | `/mnt/app` | QNX6 Power-Safe, 16 KiB blocks | `tools/qnx6_extract.py` (extraction tool, not in this repository) |
| `efs-system.img` | `/mnt/system` | QNX FFS3 (`QSSL_F3S`) | `tools/ffs3_extract.py` (extraction tool, not in this repository) |
| `main_stage2.img` / `img_restore/main_stage2.ifs.lzo` | IFS (`/`, `/proc/boot`, `/ifs`) | Alpine `LZ4_` container holding six QNX IFS images | `tools/mh2p_unlz4_ifs.py` (extraction tool, not in this repository), then `dumpifs -x` |

The extraction tools are not part of this repository. Format notes:

- QNX6 with 16 KiB blocks: data block 0 starts at the first block boundary after the superblock area
  (0x4000), not at 0x3000.
- FFS3: 128 KiB units, 32-byte extent headers growing down from each unit's end, text offsets in 64-byte
  units, logical unit n = physical unit n-1.
- `LZ4_` container: magic, u32 chunk size (2 MiB), u32 compressed sizes until 0, a `PART`/`SIGN`
  signature block at 0x400, data at 0x2000. Each chunk is an independent raw LZ4 block **preceded by 0xFF
  padding that the size field does not count**.
- The SWDL flash image `main_stage2.img` (31 MiB) is only a prefix of the same 71 MB stream and does not
  decode completely. The complete copy is `img_restore/main_stage2.ifs.lzo` inside `app.img`.

## Relationship to the other baselines

| | MHI2 (this repo) | MHI2Q | MH2p (this folder) |
| --- | --- | --- | --- |
| Firmware | `MU0678-MHI2_ER_AUG22_P3241` | `MHI2Q_ER_AUG22_P5092`, MU1329 | `MH2p_ER_AUG33_P2873` |
| OS | QNX (version not recorded) | QNX 6.5.0 SP1 | QNX 6.6.0 |
| SoC | not recorded (the `tegra3` USB device-controller drivers suggest Tegra 3) | **Qualcomm APQ8064** (`qcbl`, `*.mbn`, `lib*_8064.so`; see [portability.md](portability.md) P-470) | NVIDIA Tegra K1 |
| Wi-Fi / BT | Marvell 88W8787, `uap0` | Marvell 88W8787, `uap0` | Broadcom BCM4359 |
| iAP2 stack | `ipod-drvr-iap2.so` + `libiap2client.so`, `/dev/ipod0` | Cinemo NME, `iap://ffs:///dev/otg-cinemo` | Cinemo NME + `libEsoIAPTransport.so` + `iap2connectionmanager` |
| AirPlay | `libairplay.so` (version not recorded here) | `libairplay.so` 210.81 | `libairplay.so` **320.17.1** |
| Wireless CarPlay | no | no | **yes (shipped)** |

The MHI2Q column is from a separate analysis of the MU1329 firmware. On the software side MH2p is much closer to MHI2Q than to MU0678: both use
Cinemo, `dio_manager` and `smartphone_integrator`. MU0678 uses the older `ipod-drvr-iap2` stack.
