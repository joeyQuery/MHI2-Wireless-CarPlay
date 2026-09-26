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
| MU/software family | MU0678-class |
| Hardware variant | Record exact variant when known |
| Firmware/software version | Record exact version |
| Source | Record dump/source origin |
| Extraction date | Record when known |
| Archive/hash | Record SHA-256 or equivalent |
| Architecture | Record exact CPU/ABI when verified |
| Modifications | Stock / modified, with details |
| Runtime correspondence | Whether runtime observations came from this exact baseline |

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

## Firmware Evidence Rule

When a conclusion depends on a dump, identify the exact dump/binary/configuration from which it was derived. If that provenance is unavailable, classify the conclusion accordingly rather than assuming the source is identical to the current target.
