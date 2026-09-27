# TRACE-006 — Bluetooth / Wi-Fi / CarPlay Session Correlation

**Status:** Target

## Objective

Establish whether Bluetooth bootstrap, Wi-Fi association, iAP2, mDNS and AirPlay events belong to the same production phone/session.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | End-to-end phone/session identity correlation |
| Process / binary | Cross-process: Bluetooth, iAP2, WLAN, mDNS, DIO, AirPlay |
| Caller / callee | No complete production cross-system edge chain recovered |
| Arguments | Not applicable until correlation trace exists |
| Return / error behaviour | Not applicable |
| IPC / ASI / DSI boundary | Cross-process/session boundaries unresolved |
| Device / socket / file boundary | Bluetooth identity, Wi-Fi association/IP and AirPlay session identifiers must be joined |
| Protocol event | HCI/iAP2/Wi-Fi/mDNS/AirPlay correlation required |
| Runtime confirmation | Target only; no timestamp-correlated production session trace committed |
| Evidence IDs | None — target trace; no completed correlation evidence yet |
| Remaining uncertainty | Identity join across all transports and session lifecycle |

Establish that Bluetooth bootstrap, Wi-Fi association, iAP2 events, mDNS discovery and AirPlay session creation belong to the same phone/session.

## Required chain

```text
Bluetooth device identity
        |
        +--> iAP2 identity/event
        |
        +--> Wi-Fi association/client identity
                    |
                    +--> IP address
                    |
                    +--> mDNS discovery
                              |
                              +--> AirPlay session
                                        |
                                        +--> DIO CarPlay session
```

## Current evidence

The individual subsystems are established, but the repository does not yet contain a single timestamped production trace correlating all of these events.

Therefore this document intentionally does not claim the chain is proven.

## Required evidence

Correlate, from one controlled session:

- Bluetooth device address/identity;
- HCI/service events;
- iAP2 events;
- Wi-Fi association and assigned address;
- mDNS query/response/registration;
- DIO session events;
- AirPlay SETUP/session identifiers;
- disconnect/finalization.

## Related evidence

The following entries provide subsystem context only; they do **not** prove that the events belong to the same phone or production session:

- E-021 — Bluetooth active-device callback reaches `CIapBTChannel::updateiAPDevice()`.
- E-022 — Bluetooth iAP endpoint reaches `open64()`.
- E-026 — AirPlay Bonjour registration uses its `interfaceName` field to select an interface index.
- E-029 — `mdnsd` explicitly consumes `MDNS_DIRECTLINK_IFACE`.
- E-042 — DIO contains a Bluetooth smartphone integration boundary with explicit CarPlay mode handling.

These contextual findings identify the subsystem boundaries that a future timestamp-correlated session trace must join; none establishes the required cross-transport identity correlation.


## Completion criterion

The trace becomes Complete only when the same phone/session can be followed across the entire chain without an inferred identity join.
