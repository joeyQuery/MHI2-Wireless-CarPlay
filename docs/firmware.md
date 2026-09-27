# MHI2 Firmware Provenance

This document defines the firmware baseline used by the reverse-engineering project.

## Analysis Target

```text
Audi MHI2
MU0678-class firmware
QNX
Marvell 8787 WLAN/BT platform
```

The exact firmware build, hardware variant and extraction provenance must be recorded for any new dump used as primary evidence.

## Required Provenance

| Field | Value |
|---|---|
| Head-unit/platform | Audi MHI2 |
| MU/software family | MU0678-class; exact analysis firmware identifier recorded below |
| Hardware variant | UNKNOWN |
| Firmware/software version / identifier | `MU0678-MHI2_ER_AUG22_P3241-JTB-07429.06.161184010D` |
| Analysis source path | `MHI2-CarPlay-Alternate-Screen/dump/MU0678-MHI2_ER_AUG22_P3241-JTB-07429.06.161184010D/advanced/MU0678-appimg` |
| Extraction date | UNKNOWN |
| Archive/hash | UNKNOWN |
| Architecture / CPU / ABI | UNKNOWN |
| Modifications | UNKNOWN for the source dump; individual modified experimental files are documented separately when applicable |
| Runtime correspondence | UNKNOWN / not formally established for all historical runtime observations |

## Current Provenance State

The current primary analysis dump is identified by the exact firmware/dump name:

```text
MU0678-MHI2_ER_AUG22_P3241-JTB-07429.06.161184010D
```

Its repository analysis source is:

```text
MHI2-CarPlay-Alternate-Screen/dump/MU0678-MHI2_ER_AUG22_P3241-JTB-07429.06.161184010D/advanced/MU0678-appimg
```

The exact identifier above is recorded from the dump name supplied for this analysis. It is **not** decoded here into separate hardware/version fields beyond what the evidence explicitly establishes. The original extraction date, archive hash, exact hardware variant, CPU/ABI, stock/modified state of the source dump, and formal one-to-one correspondence between every historical runtime observation and this exact image remain UNKNOWN unless separately documented.

The repository therefore has a known exact analysis identifier and source path, but incomplete forensic provenance. Provenance-dependent conclusions must remain tied to their stated source documents and must not be promoted merely because they appear in multiple documents.

This is a documentation blocker, not a reason to invent missing values.

## Baseline Rule

Static evidence and runtime evidence must not be silently mixed across firmware versions.

If a symbol, path, configuration value or address comes from one image while runtime behaviour comes from another, the relationship must be explicitly labelled.

## Modified Images

Modified binaries used for experiments must retain:

- original filename/path;
- original hash;
- modified hash;
- modification description;
- experiment identifier;
- rollback status.

Large binary artifacts should not be committed to this repository unless there is a specific reason to do so.

## Known Firmware Components

The repository has established evidence for components including:

```text
dio_manager
libairplay.so
bluetooth
btstack
connectionmanager
libiap2client.so
ipod-drvr-iap2.so
mss-ipodiap2.so
devu-iap2-tegra3-ci.so
devu-iap2ncm-tegra3-ci.so
devnp-usbdnet.so
io-sdiorm-mib2
devnp-mrvl_wlan-sdiorm.so
uaputl
dnsmasq
```

This is a component inventory, not a claim that all components participate in Wireless CarPlay.

## Provenance Completion Checklist

Before treating a new dump as a primary evidence baseline, record:

```text
exact software/firmware version
exact hardware variant
source/origin of dump
extraction date
SHA-256 (or equivalent)
CPU/ABI
stock vs modified state
relationship between dump and runtime unit
```

If a field is unknown, write `UNKNOWN` rather than leaving provenance ambiguous. The exact analysis identifier above is known; do not infer additional hardware, extraction or ABI facts from its name alone.

## Firmware Evidence Rule

When a conclusion depends on a dump, identify the exact dump/binary/configuration from which it was derived. If that provenance is unavailable, classify the conclusion accordingly rather than assuming the source is identical to the current target.
