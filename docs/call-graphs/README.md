# Function-Level Call Graphs

This directory contains the function/process call-graph layer of the Wireless CarPlay investigation.

## Purpose

Subsystem documents describe **what exists**.

The evidence register records **why a conclusion is accepted**.

Call graphs describe **how execution actually moves through the MHI2 binaries**.

## Required Format

Each completed call graph should record:

```text
Entry point
Process / binary
Function / symbol
Address, when known
Caller
Callee
Arguments
Return value / error handling
IPC or transport boundary
Device/socket/file boundary
Runtime confirmation
Evidence IDs
Remaining uncertainty
```

## Priority Graphs

### Bluetooth / iAP2

```text
enableIap
  ↓
Bluetooth iAP registration/startup
  ↓
Bluetooth iAP proxy
  ↓
iAP2 transport
  ↓
DIO
```

These arrows are investigation targets, not a completed trace.

### DIO / iAP2

```text
notifyiAP2DeviceConnected
  ↓
iAP2Connect
  ↓
CIpodAP2Service
  ↓
/dev/ipod0 or transport abstraction
```

Recover the actual caller/callee sequence and arguments.

### DIO / AirPlay

```text
DIO
  ↓
AirPlayReceiverServer
  ↓
AirPlayReceiverSession
  ↓
AirPlayReceiverSessionScreen
  ↓
SetIFName / SetTransportType / SetClientIfMACAddr
```

Recover the exact runtime argument values.

### mDNS / direct-link

```text
MDNS_DIRECTLINK_IFACE
  ↓
unknown consumer
  ↓
socket/interface selection
  ↓
AirPlay / Bonjour
```

Recover the consumer before changing the configuration.

## Relationship to Execution Traces

The call-graph directory defines the function-level record format. Completed transport-boundary traces are maintained in [`traces/`](../../traces/README.md). A call-graph target is not promoted to a trace edge until the caller/callee relationship is actually recovered.

## Evidence Standard

A call graph is complete only when its important edges are supported by binary analysis, runtime evidence, protocol evidence, or a clearly labelled inference.

Do not turn a conceptual architecture arrow into a call-graph edge without evidence.
