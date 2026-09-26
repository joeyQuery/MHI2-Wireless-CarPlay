# TRACE-006 — Bluetooth / Wi-Fi / CarPlay Session Correlation

**Status:** Target

## Objective

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

- E-015 — Wireless CarPlay is not yet proven end-to-end

## Completion criterion

The trace becomes Complete only when the same phone/session can be followed across the entire chain without an inferred identity join.
