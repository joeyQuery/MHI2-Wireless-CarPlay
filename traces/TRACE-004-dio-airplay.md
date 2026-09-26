# TRACE-004 — DIO → AirPlay

**Status:** Partial

## Objective

Recover the actual DIO-to-AirPlay session construction and interface/transport arguments.

## Established

DIO references:

```text
AirPlayReceiverServer
AirPlayReceiverSession
AirPlayReceiverSessionSetup
AirPlayReceiverSessionScreen
AirPlayReceiverSessionChangeModes
AirPlayReceiverSessionSetSecurityInfo
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
```

DIO also contains CarPlay session/event symbols.

## Current trace

```text
DIO
 |
 +--> AirPlayReceiverServer
       |
       +--> AirPlayReceiverSession
              |
              +--> Setup
              +--> Screen
              |      |
              |      +--> SetIFName [argument unresolved]
              |      +--> SetTransportType [argument unresolved]
              |      +--> SetClientIfMACAddr [argument unresolved]
              |
              +--> Security / mode handling
```

The integration boundary is established, but the exact callers, arguments, timing and return/error handling are not.

## Evidence

- E-009 — AirPlay interface-selection helpers
- E-010 — DIO AirPlay/CarPlay integration
- E-015 — Wireless CarPlay is not yet proven end-to-end

## Required next trace

Recover each setter's caller and arguments, then correlate them with the session's network interface and client identity.

## Decision gate

Do not patch `libairplay.so` merely to force Wi-Fi. The DIO-supplied values must be recovered first.
