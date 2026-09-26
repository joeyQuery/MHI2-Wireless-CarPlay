# TRACE-004 — DIO → AirPlay

**Status:** Partial

## Objective

Recover the actual DIO → AirPlay session construction path and the interface/transport arguments supplied to the screen APIs.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | DIO AirPlay receiver/session integration |
| Process / binary | `dio_manager`; `libairplay.so` |
| Caller / callee | AirPlay receiver/session symbols are established; exact caller/callee sequence unresolved |
| Arguments | `SetIFName`, `SetTransportType`, `SetClientIfMACAddr` values unresolved |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Unresolved |
| Device / socket / file boundary | Network interface/client MAC boundary unresolved |
| Protocol event | AirPlay SETUP/session relationship is not yet correlated to the interface setters |
| Runtime confirmation | Integration symbols are present; complete runtime call sequence is not |
| Evidence IDs | E-009, E-010 |
| Remaining uncertainty | Exact callers, arguments, timing and error handling |

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

## Required next trace

Recover each setter's caller and arguments, then correlate them with the session's network interface and client identity.

## Decision gate

Do not patch `libairplay.so` merely to force Wi-Fi. The DIO-supplied values must be recovered first.
