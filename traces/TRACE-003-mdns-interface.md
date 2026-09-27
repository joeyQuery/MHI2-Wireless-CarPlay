# TRACE-003 — MDNS_DIRECTLINK_IFACE

**Status:** Partial

## Objective

Recover where `MDNS_DIRECTLINK_IFACE=carplay0` is consumed and how the value reaches mDNS/AirPlay network binding.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | `MDNS_DIRECTLINK_IFACE` configuration |
| Process / binary | `mdnsd`; `dio_manager` contains the production configuration value |
| Caller / callee | `mdnsd::SetupInterfaceList()` → `SetupOneInterface()` → `getenv("MDNS_DIRECTLINK_IFACE")`; boot-time producer remains unresolved |
| Arguments | Interface value is `carplay0` in production configuration; downstream arguments unresolved |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Process boundary between AirPlay/DNS-SD and `mdnsd` is established; environment-variable provenance into `mdnsd` remains unresolved |
| Device / socket / file boundary | `carplay0` is production configuration; `mdnsd` has dedicated direct-link interface registration; exact socket path remains unresolved |
| Protocol event | Bonjour/mDNS APIs are present; production CarPlay discovery path on the configured interface is not fully traced |
| Runtime confirmation | Consumer logic is statically proven in `mdnsd`; boot-time environment assignment is not runtime-proven |
| Evidence IDs | E-003, E-004, E-008, E-009, E-017, E-028, E-029, E-030 |
| Remaining uncertainty | Who exports `MDNS_DIRECTLINK_IFACE` into the running `mdnsd` environment and the final socket-binding details |

Recover where `MDNS_DIRECTLINK_IFACE=carplay0` is read and how its value reaches the mDNS/AirPlay network path.

## Newly recovered binary trace

In production `dio_manager`, configuration metadata contains:

```text
MDNS_DIRECTLINK_IFACE=carplay0
```

at virtual address `0x190d2c`, adjacent to other mDNS configuration metadata. This establishes the value as part of the binary's configuration data rather than an isolated architectural description.

The mDNS consumer is now recovered: `mdnsd` explicitly calls `getenv("MDNS_DIRECTLINK_IFACE")` from `SetupOneInterface()`, retains the interface name in its interface structure, and registers that interface with the mDNS platform. What remains unresolved is how the running `mdnsd` process receives the variable at boot. The consumer edge is therefore proven; the producer/provenance edge and final socket details remain open.

## Newly recovered mDNS trace

`mdnsd` contains `MDNS_DIRECTLINK_IFACE` and `SetupOneInterface()` explicitly obtains the variable with `getenv("MDNS_DIRECTLINK_IFACE")`. `SetupInterfaceList()` feeds interfaces into `SetupOneInterface()`, which stores the interface name in its interface structure and continues into mDNS interface registration. This proves the direct-link consumer mechanism inside `mdnsd`.

It does **not** prove that the production boot sequence exports `MDNS_DIRECTLINK_IFACE=carplay0` into `mdnsd`'s environment. The occurrence in `dio_manager` is configuration metadata, not a recovered `putenv()` callsite.

## Established

Production configuration contains:

```text
MDNS_DIRECTLINK_IFACE=carplay0
```

The production USB CarPlay network is:

```text
USB -> devnp-usbdnet.so -> carplay0
```

AirPlay contains Bonjour/mDNS APIs and explicit network-interface helpers.

## Current trace

```text
DIO / production configuration
        |
        | MDNS_DIRECTLINK_IFACE=carplay0
        X  boot-time environment export not yet proven
        |
        v
mdnsd
  |
  +--> SetupInterfaceList()
  |       |
  |       v
  |   SetupOneInterface()
  |       |
  |       +--> getenv("MDNS_DIRECTLINK_IFACE")
  |       |
  |       +--> retain interface name
  |       |
  |       +--> register interface with mDNS platform
  |
  +--> [final socket binding details: unresolved]
```

The configuration value itself is proven. The downstream chain is not.

## Additional recovered negative

The production /etc/scripts/mdnsd.sh script only recreates and chmods /var/run/mdnsd. It contains no environment assignment/export for MDNS_DIRECTLINK_IFACE. This rules out that script as the recovered boot-time producer, but does not identify the actual producer elsewhere in the boot/configuration chain.

**Evidence:** E-038.

## Evidence

- E-028 — `libairplay` DNS-SD calls form the AirPlay → `libdns_sd` boundary
- E-029 — `mdnsd` explicitly reads `MDNS_DIRECTLINK_IFACE` via `getenv()`
- E-030 — `mdnsd` implements dedicated direct-link interface registration

- E-004 — production `MDNS_DIRECTLINK_IFACE=carplay0`
- E-008 — AirPlay exposes Bonjour/mDNS APIs
- E-009 — AirPlay exposes interface-selection helpers

## Required next trace

Search every consumer/reference, identify parser and process, recover the variable/object carrying the interface name, then connect it to socket setup or AirPlay configuration.

## Decision gate

Do not conclude that changing the value to `uap0` is sufficient until the consumer and downstream binding are known.
