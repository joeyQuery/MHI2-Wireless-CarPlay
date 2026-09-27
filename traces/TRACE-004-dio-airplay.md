# TRACE-004 — DIO → AirPlay

**Status:** Partial

## Objective

Recover the actual DIO → AirPlay session construction path and the interface/transport arguments supplied to the screen APIs.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | DIO AirPlay receiver/session integration |
| Process / binary | `dio_manager`; `libairplay.so` |
| Caller / callee | DIO creates the production AirPlay server; exact session caller/argument sequence remains unresolved |
| Arguments | Screen setter arguments are not usable as the presumed transport-selection path because the production setters are no-op stubs; Bonjour interfaceName flow is recovered inside `libairplay` |
| Return / error behaviour | Unresolved |
| IPC / ASI / DSI boundary | Unresolved |
| Device / socket / file boundary | AirPlay Bonjour registration uses an interface index derived from the server/object `interfaceName`; lower-level packet/multicast socket path remains unresolved |
| Protocol event | AirPlay SETUP/session relationship is not yet correlated to the interface setters |
| Runtime confirmation | Integration symbols are present; complete runtime call sequence is not |
| Evidence IDs | E-009, E-010, E-018, E-019, E-025, E-026, E-027 |
| Remaining uncertainty | Exact DIO session call sequence, how `interfaceName` is populated, packet/multicast binding callers and runtime selected interface |

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

DIO creates the production AirPlay server through `AirPlayReceiverServerCreate(...)`, rather than the config-file creation variant. The production `libairplay.so` contains `_UpdateBonjourAirPlay`, where the AirPlay object has an `interfaceName` field at `object + 0x6c`. If non-empty, the code calls `if_nametoindex(interfaceName)` and passes the resulting interface index into `DNSServiceRegister(...)`; otherwise the index is `0`.

This is the concrete interface-selection path currently recovered inside AirPlay. It supersedes the earlier hypothesis that the exported screen setters were necessarily the active transport-selection mechanism.

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
 +--> AirPlayReceiverServerCreate(...)
       |
       +--> AirPlayReceiverServer / session
              |
              +--> _UpdateBonjourAirPlay
              |      |
              |      +--> object + 0x6c: interfaceName
              |      |
              |      +--> if_nametoindex(interfaceName)
              |      |
              |      +--> DNSServiceRegister(..., interfaceIndex, ...)
              |
              +--> Screen setter exports
                     |
                     +--> SetIFName [production no-op]
                     +--> SetTransportType [production no-op]
                     +--> SetClientIfMACAddr [production no-op]
```

The lower-level packet/multicast socket path is traced separately in TRACE-005.


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

- E-025 — DIO creates the production AirPlay server with `AirPlayReceiverServerCreate(...)`
- E-026 — AirPlay `interfaceName` is converted with `if_nametoindex()` and supplied to `DNSServiceRegister()`
- E-027 — screen interface/transport/client-MAC setter exports are no-op stubs in production

- E-009 — AirPlay interface-selection helpers
- E-010 — DIO AirPlay/CarPlay integration

## Required next trace

Recover where the AirPlay object's `interfaceName` field is populated, trace callers of the substantive packet/multicast interface helpers, and correlate Bonjour registration with the AirPlay SETUP/session and DIO session.

## Decision gate

Do not patch `libairplay.so` merely to force Wi-Fi. The DIO-supplied values must be recovered first.
