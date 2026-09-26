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
| Arguments | Caller values unresolved; production implementations of the three exported screen setters are no-op stubs |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Unresolved |
| Device / socket / file boundary | Network interface/client MAC boundary unresolved |
| Protocol event | AirPlay SETUP/session relationship is not yet correlated to the interface setters |
| Runtime confirmation | Integration symbols are present; complete runtime call sequence is not |
| Evidence IDs | E-009, E-010, E-018 |
| Remaining uncertainty | Exact callers, arguments, timing and error handling |

Recover the actual DIO-to-AirPlay session construction and interface/transport arguments.

## Newly recovered binary trace

In pristine production `libairplay.so`, these exported APIs are present:

```text
AirPlayReceiverSessionScreen_SetClientIfMACAddr  0x25bd0
AirPlayReceiverSessionScreen_SetIFName           0x25bdc
AirPlayReceiverSessionScreen_SetTransportType    0x25be0
```

Their production implementations are effectively:

```asm
bx lr
```

This corrects the earlier interpretation that their presence represented active transport-selection machinery. Their symbols are real, but these three entry points do not perform observable work in this build.

The DIO caller/argument path remains unresolved, and these setters should no longer be treated as the presumed USB-to-Wi-Fi adaptation point.

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
