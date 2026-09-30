# TRACE-013 — MU0678 Bluetooth iAP2 → DIO / AirPlay Boundary

**Status:** Partial

## Objective

Consolidate the latest production-binary trace of the Bluetooth iAP2 path and identify the remaining binary-level boundary before Wireless CarPlay can be considered end-to-end proven.

## Firmware baseline

- Platform: Audi MHI2 / MU0678-class QNX
- Production image: MU0678 application image used throughout the repository
- Relevant binaries:
  - `eso/bin/apps/iap`
  - `eso/bin/apps/dio_manager`
  - `eso/lib/factories/libasimmxconnectivity_bluetooth_iapproxy.so`
  - `eso/lib/libairplay.so`
  - `eso/bin/apps/mdnsd`

## 1. Bluetooth iAP2 is a production subsystem

The production `iap` binary contains a dedicated Bluetooth transport implementation:

```text
iap::CIapBTChannel
  openiAPDevice()
  connectToiAPDevice()
  readiAPDevice()
  writeiAPDevice()
  updateiAPDevice()
  closeiAPDevice()
```

It also contains `CIapConnectorMachine`, including iAP support checks and state transitions, plus iAP2 SYN/SYNACK/ACK/link-packet machinery.

The Bluetooth path is therefore more than pairing or dormant capability.

**Evidence:** E-014, E-015, E-021, E-022.

## 2. Bluetooth iAP uses an RPC/service boundary

The production image contains:

```text
asi.connectivity.bluetooth.iap.IapDeviceServices
```

in `libasimmxconnectivity_bluetooth_iapproxy.so`, with matching service/reply machinery in `iap`.

The recovered architecture is:

```text
Bluetooth connectivity service
        |
        | COMM/RPC
        v
IapDeviceServices
        |
        v
iap / CIapBTChannel
        |
        +--> open64()
        +--> read()
        +--> write()
        +--> close()
```

The exact endpoint publication/consumer path remains unresolved.

**Evidence:** E-015, E-021, E-022.

## 3. The Bluetooth endpoint is not proven to be /dev/ipod0

`CIapBTChannel::openiAPDevice()` reaches `open64()` with a runtime-supplied endpoint. No `/dev/ipod0` literal was recovered from the `iap` binary.

Separately, DIO's production USB iAP2 path is:

```text
dio_manager
  -> CIpodAP2Service
  -> iap2_connect()
  -> configured /dev/ipod0
```

This proves a transport/path separation. It does **not** prove that the Bluetooth endpoint is incompatible with the iAP2 resource-manager ABI.

**Evidence:** E-016, E-022, E-023, E-049, E-050, E-051, E-054, E-055.

## 4. DIO/AirPlay network boundary

DIO contains production AirPlay lifecycle integration and the configuration value:

```text
MDNS_DIRECTLINK_IFACE=carplay0
```

The value is part of DIO configuration. The existing `carplay0` path is the USB CarPlay network interface.

The production `libairplay.so` also contains real:

```text
SocketSetPacketReceiveInterface
SocketSetMulticastInterface
IsWiFiNetworkInterface
DNSService*
```

network machinery.

However, the following exported screen setters are production no-op stubs and must not be treated as the active wireless-interface selector:

```text
AirPlayReceiverSessionScreen_SetIFName
AirPlayReceiverSessionScreen_SetTransportType
AirPlayReceiverSessionScreen_SetClientIfMACAddr
```

**Evidence:** E-017, E-018, E-019, E-020, E-025, E-026, E-027, E-028, E-029, E-030.

## 5. Current recovered architecture

The MU0678-specific evidence establishes two separate branches:

```text
Bluetooth
   |
   v
IapDeviceServices COMM/RPC
   |
   v
iap / CIapBTChannel
   |
   v
runtime-supplied endpoint
   |
   +---- [MU0678 endpoint owner unresolved]
```

and independently:

```text
DIO
   |
   +---- iAP2 client -> configured /dev/ipod0
   |
   +---- AirPlay server / Bonjour / mDNS
```

These branches must **not** currently be joined as:

```text
Bluetooth endpoint -> DIO /dev/ipod0
```

The production evidence does not establish that edge.

### MH2p reference correction

The merged MH2p reference provides a concrete alternative architecture: its Bluetooth iAP2 endpoint is consumed by a separate bootstrap client (`iap2connectionmanager`/SI), while the wireless CarPlay session is started in DIO with Wi-Fi connection information and DIO's session-side iAP2 control travels through AirPlay (`iAPSendMessage`). This is **cross-platform reference evidence only** and does not prove MU0678 implements the same path.

Therefore the useful target for MU0678 is now:

```text
Bluetooth bootstrap
      |
      v
  iAP2 client
      |
      +---- phone identification / Wi-Fi bootstrap
      |
      v
Wi-Fi AP (uap0)
      |
      v
  AirPlay / mDNS
      |
      v
     DIO
      |
      +---- session-side iAP2 over AirPlay ?   <-- MU0678 feature gap to resolve
```

The question is no longer simply "how does Bluetooth reach DIO?" It is whether MU0678 contains enough of the MH2p **bootstrap + Wi-Fi AirPlay + session-side iAP2** architecture to reproduce that sequence, or whether missing receiver-side protocol functionality must be ported.

## 6. Remaining blocker

The remaining static question is now specific:

> What exact endpoint does the Bluetooth service supply to `CIapBTChannel::openiAPDevice()`, who owns that endpoint, and does it terminate at the same QNX iAP2 resource-manager ABI implemented by `ipod-drvr-iap2.so)?

The production `bluetooth` executable is the highest-value missing artifact for resolving the first half of this question. Without it, the endpoint construction/ownership branch must remain unresolved.

A runtime capture of the endpoint would also resolve the ambiguity if the binary remains unavailable.

## 7. AirPlay follow-up

The network-side investigation should continue independently of the Bluetooth endpoint:

1. Recover where the AirPlay object's `interfaceName` is populated.
2. Trace callers/arguments to `SocketSetPacketReceiveInterface`.
3. Trace callers/arguments to `SocketSetMulticastInterface`.
4. Establish whether the resulting path can operate on `uap0` without modification.
5. Keep `MDNS_DIRECTLINK_IFACE=carplay0` classified as the current production configuration, not as proof that `uap0` is already selected.

## Evidence summary

- **Proven:** production Bluetooth iAP2 implementation and RPC boundary.
- **Proven:** Bluetooth endpoint is runtime-supplied to `open64()`.
- **Proven:** DIO's current configured iAP2 path is `/dev/ipod0`.
- **Proven:** AirPlay has substantive Wi-Fi/interface/socket primitives.
- **Disproven/corrected:** the three screen interface/transport setters are not active selectors in this production build.
- **Partial:** Bluetooth endpoint → iAP2 bootstrap/client handoff; a direct Bluetooth endpoint → DIO handoff is not established and should not be assumed.
- **Partial:** DIO/AirPlay runtime interface selection.
- **Not yet proven:** end-to-end Wireless CarPlay.

## Related evidence

E-014 through E-031, E-049 through E-055.

## Related traces

- TRACE-001 — Bluetooth → iAP2
- TRACE-002 — DIO → iAP2 transport / path-driven client boundary
- TRACE-003 — MDNS_DIRECTLINK_IFACE
- TRACE-004 — DIO → AirPlay
- TRACE-005 — AirPlay → socket/interface
- TRACE-008 — iAP2 control plane / DIO Bluetooth integration
- TRACE-011 — DIO runtime iAP2 endpoint
- TRACE-012 — iAP2 client → driver resource-manager boundary

## 8. New MU0678 configuration/deployment trace

The exact MU0678 application-image tree independently proves that btstack is a separately supervised production process:

```text
eso/production/connectivity.json

applications:
  bluetooth  -> /eso/bin/apps/bluetooth
  btstack    -> /eso/bin/apps/btstack

btstack:
  exec: /eso/bin/apps/btstack
  preCondition: /tmp/mvloaded
```

The same production configuration contains:

```text
bluetooth:
  topologyLogic: 1
  enableIap: false
```

This establishes an important separation: enableIap=false is a setting under the bluetooth application configuration, while the btstack process is independently launched when /tmp/mvloaded exists. Therefore enableIap cannot be described merely as the switch that starts btstack or creates its RFCOMM service.

The shipped mm/iap2.cfg independently selects:

```text
[transport]
name=Lightning Connector
id=1234
```

Its commented [bluetooth] section is explicitly documented as Bluetooth Connection Status handling (mac, connectstatus, etc.), not as selection of the underlying iAP2 transport. This closes the interpretation that uncommenting that section is itself the production transport selector.

The exact MU0678 dump also contains eso/bin/apps/btstack (1,912,117 bytes; blob d9309fc964aa4c6dbe92c180d8ba0dee61899328). The available repository binary-reading interface cannot decode this non-UTF-8 ELF, so its internal endpoint publisher still cannot be claimed from direct binary inspection.

## 9. New negative evidence against bluetooth as the endpoint publisher

The locally recovered MU0678 bluetooth ELF was inspected directly. Its defined C++ functions include CBluetoothApplication and CBluetoothSmartphoneIntegration, but no defined C++ CBluetoothIap* / iAP endpoint-construction / RFCOMM-server function symbols were recovered. Its dynamic dependencies are limited to the production framework libraries (libdsicommon, libiplcommon, libosal, libutil, libcomm, libirc_mmx_adapter, libecpp-ne, libc, libm). No direct undefined iAP/RFCOMM endpoint API symbols were recovered either.

The binary does contain the policy/service-level strings iapEnabled, bluetooth.enableIap, SERVICETYPE_IAP2, ERROR_CARPLAY_ACTIVE, Carplay, and RFCOMM error handling. That is evidence for Bluetooth/iAP policy integration, but not evidence that this executable itself creates the concrete iAP endpoint.

This moves the endpoint-owner hypothesis downward one layer: the highest-value unresolved owner remains btstack or another lower Bluetooth/service component, with IapDeviceServices as the publication/consumption boundary already proven in iap.

## 12. MU0678 `bluetooth` ELF is now directly inspected

The production `/eso/bin/apps/bluetooth` ELF is now available for direct local inspection, so the previous repository-access limitation is retired.

Concrete findings:

- The binary contains `iapEnabled` and `bluetooth.enableIap` strings.
- It contains `SERVICETYPE_IAP2`, `ERROR_CARPLAY_ACTIVE`, `Carplay`, and RFCOMM-related error text.
- It defines `CBluetoothApplication` and `CBluetoothSmartphoneIntegration` service/vtable objects.
- No recovered defined `CBluetoothIap*`, concrete iAP endpoint-construction, or RFCOMM-server function symbols were found.
- No direct undefined iAP/RFCOMM endpoint API imports were recovered.
- `CBluetoothApplication::switchBluetoothAccordingToConfig()` exists at `0x13df18` and is called from `CBluetoothApplication::diagCBCodingValues(...)` at `0x146500`.
- That function checks several application-state byte fields, including offsets `0x1182`, `0x1186`, and `0x118a`, but those fields have **not** been proven to be `bluetooth.enableIap`.

This is useful negative evidence: the concrete Bluetooth endpoint publisher is not exposed as an obvious direct iAP/RFCOMM API inside this executable. It narrows the remaining ownership search toward the service/proxy path and `btstack`, but does not prove `btstack` is the owner.

**Evidence:** E-068, E-069.

## Updated remaining blocker


The remaining static question is now narrower:

> Which production component creates/publishes the runtime endpoint consumed through `IapDeviceServices`, and what exact QNX resource-manager node and iAP2 ABI does it expose?

`btstack` remains the highest-value candidate because it is independently supervised and the inspected `bluetooth` ELF does not expose an obvious concrete endpoint-construction API. That is a search priority, **not yet a proven ownership edge**.

If btstack does not contain that machinery, the next candidate must be another lower Bluetooth/service component; no evidence currently permits naming one.

## 10. Corrected DIO → AirPlay import/call trace

Inspection of the pristine MU0678 `dio_manager` ELF establishes this split:

- `dio_manager` imports `AirPlayReceiverServerCreate` and has a recovered direct call at `0x13f0b4`.
- `dio_manager` imports `AirPlayReceiverServerSetDelegate` and has a recovered direct call at `0x13f34c`.
- `AirPlayReceiverServerSetProperty` is present as a dynamic symbol, but its relocation is `R_ARM_GLOB_DAT` at `0x19da70`, not a normal PLT/JUMP_SLOT entry. I have **not** recovered a direct callsite to that symbol in the current static pass.
- `dio_manager` does **not** import `SocketSetPacketReceiveInterface`, `SocketSetMulticastInterface`, `SocketSetBoundInterface`, or `IsWiFiNetworkInterface`.
- `dio_manager` does import `CFObjectSetPropertyCString`, with direct callsites, but those calls have not been proven to set the AirPlay server's `interfaceName` property.

The earlier claim that a direct `AirPlayReceiverServerSetProperty` callsite had been recovered was therefore incorrect and is retired.

The valid DIO-side trace currently stops at:

```text
dio_manager
  |
  +--> AirPlayReceiverServerCreate()       @ 0x13f0b4
  |
  +--> AirPlayReceiverServerSetDelegate()  @ 0x13f34c
  |
  +--> [server property population: unresolved]
```

The next static target is the GLOB_DAT-based use of `AirPlayReceiverServerSetProperty`, or the object/property construction path that ultimately populates AirPlay's `interfaceName`.

## 11. Current AirPlay boundary

```text
dio_manager
  |
  +--> AirPlayReceiverServerCreate()       <-- direct call proven
  +--> AirPlayReceiverServerSetDelegate()  <-- direct call proven
  +--> AirPlayReceiverServerSetProperty()  <-- imported; direct caller unresolved
  |
  X--> SocketSetPacketReceiveInterface()    <-- not imported by DIO
  X--> SocketSetMulticastInterface()        <-- not imported by DIO
  X--> IsWiFiNetworkInterface()             <-- not imported by DIO
  X--> Screen_SetIFName/SetTransportType()  <-- not imported by DIO
```

Do not infer from `uap0`, `carplay0`, or the string `interfaceName` that either interface is selected. The active property-setting edge remains unresolved.

## 12. Full GLOB_DAT trace: `AirPlayReceiverServerSetProperty` is active, but this call is not the wireless-interface setter

The MU0678 `dio_manager` GLOB_DAT relocation at `0x19da70` is not an unused import. A complete static path was recovered:

```text
dio_manager .got
  0x19da70 = AirPlayReceiverServerSetProperty
       ^
       | GOT base + 0x648
       |
function around 0x13f0xx
  ldr r7, [pc, ...]       -> 0x648
  ldr r2, [r4, r7]        -> GOT[0x648]
  ...
  bl 0x115a64             -> CFObjectSetPropertyCString
       |
       v
libairplay::CFObjectSetPropertyCString
       |
       +--> CFStringCreateWithBytes(...)
       |
       +--> CFObjectSetProperty(...)
                 |
                 +--> bit 0 of flags is set
                 +--> blx callback
                         callback = AirPlayReceiverServerSetProperty
```

At the first recovered callsite (`0x13f128`), the callback pointer is explicitly loaded from GOT slot `0x19da70`. A second callsite (`0x13f1f0`) uses the same path. Therefore `R_ARM_GLOB_DAT` here is a real indirect callback edge, not dead linkage.

The callsite also initializes a 17-byte temporary buffer with `memset(..., 0, 17)` and passes it through `CFObjectSetPropertyCString`; the length argument is `-1`, so the generated CFString follows the implementation's null-terminated-string path. The callback receives the server object and the property arguments through `CFObjectSetProperty`.

Inside `libairplay::AirPlayReceiverServerSetProperty` (`0x1dc88`), the function uses `CFEqual` on its third argument against three internal property objects before taking the corresponding branches. This proves the indirect callback reaches the real server-property dispatcher.

However, the property argument recovered at the MU0678 callsite is derived from a DIO read-only-data pointer and the temporary string is empty. The static pass does **not** recover the literal `interfaceName` as the property object at this callsite. Therefore this GLOB_DAT trace does not prove the DIO wireless-interface binding.

### Consequence

The earlier search target was too broad. The GLOB_DAT question is now closed at the linkage/execution level:

- `AirPlayReceiverServerSetProperty` **is actively invoked** through the GLOB_DAT slot.
- It reaches the real `libairplay` property dispatcher.
- The recovered invocation is **not sufficient evidence that `interfaceName` is being set**.
- The remaining MU0678 interface-selection question is the identity of the property object passed into the dispatcher and the separate path that would populate `interfaceName`.

Therefore the active next AirPlay target is no longer the GLOB_DAT relocation itself; it is the property-object construction/population feeding `AirPlayReceiverServerSetProperty`.

## 13. Instruction-level refinement: Bluetooth configuration gate

The locally inspected MU0678 ELF closes one more part of the configuration-control trace without changing the unresolved field-name mapping.

The caller is concrete:

```text
CBluetoothApplication::diagCBCodingValues(...)  @ 0x146500
    |
    +-- bl 0x13df18
            |
            v
CBluetoothApplication::switchBluetoothAccordingToConfig()
```

The first decision sequence in `switchBluetoothAccordingToConfig()` is:

```text
0x13df40: MOVW r3, #0x1182
0x13df44: LDRB r3, [r5, r3]
0x13df48: CMP  r3, #0
0x13df4c: BEQ  ...

0x13df50: MOVW r3, #0x1186
0x13df54: LDRB r3, [r5, r3]
0x13df58: CMP  r3, #0
0x13df5c: BNE  ...

0x13df78: MOVW r3, #0x118A
0x13df7c: LDRB r3, [r5, r3]
0x13df80: CMP  r3, #0
0x13df84: BEQ  ...
```

So the following is now **proven**:

- these are object-relative byte fields, not guessed constants;
- all three are consumed by the real Bluetooth configuration-control function;
- the function is reached from `diagCBCodingValues()` at `0x146500`;
- the fields participate in the branch that decides whether the Bluetooth reconfiguration path proceeds.

What remains **unproven** is which field corresponds to which named configuration item. In particular, the presence of the strings `iapEnabled` and `bluetooth.enableIap` elsewhere in the ELF is not sufficient to assign either string to `+0x1182`, `+0x1186`, or `+0x118A`.

This therefore narrows the next static target to the code that populates those fields from the configuration object, rather than treating `enableIap=false` as already mapped to one of them.


## 14. MU0678 `enableIap` storage and consumer trace

The direct inspection of the production MU0678 `bluetooth` ELF closes the previously missing configuration-object layer.

### 14.1 `CBluetoothTopologyReconnect` owns the `enableIap` cached byte

The vtable at `0x248208` resolves to `_ZTVN10app_con_bt27CBluetoothTopologyReconnectE`. Its constructor is at `0x155178`.

The constructor creates:

```text
object + 0x04  -> "bluetooth.disableMap" property
object + 0x14  -> cached disableMap byte
object + 0x18  -> "bluetooth.enableIap" property
object + 0x28  -> cached enableIap byte
```

Relevant instructions:

```text
0x1551d8  BL  0x215eb4
0x1551dc  STRB r10,[r4,#0x14]
0x155208  ADD  r0,r4,#0x18
0x15520c  BL   0x215eb4
0x155210  STRB r10,[r4,#0x28]
```

The application construction path at `0x1475d8` passes the in-place member at `application + 0xB8` into this constructor:

```text
0x1475e4  ADD r6,r0,#0xB8
0x1475ec  BL  0x155178
```

Therefore:

```text
CBluetoothApplication
    +0xB8 -> CBluetoothTopologyReconnect
                 +0x18 -> enableIap property
                 +0x28 -> cached enableIap byte
```

This definitively separates `bluetooth.enableIap` from the `+0x1182/+0x1186/+0x118A` vehicle-coding validity fields.

### 14.2 The cached byte has a concrete consumer

The first direct byte read of this `+0x28` member outside constructor/destructor bookkeeping is:

```text
0x17b29c  LDRB r3,[r5,#0x28]
```

It is inside helper `0x17b280`, which begins with `r5 = r0` and then compares the loaded byte:

```text
0x17b288  MOV  r5,r0
0x17b29c  LDRB r3,[r5,#0x28]
0x17b2a4  CMP  r3,#0
```

The helper has two direct callers:

```text
0x17b598  BL  0x17b280
0x17baf0  MOV r0,r4
0x17baf4  BL  0x17b280
```

Thus the cached configuration byte is not merely stored: it is read and participates in a real control-flow decision.

### 14.3 Provenance boundary

**Proven:** `bluetooth.enableIap` → `CBluetoothTopologyReconnect +0x28`, with the topology-reconnect object constructed at `CBluetoothApplication +0xB8`.

**Proven:** a live consumer exists at `0x17b29c` through helper `0x17b280`.

**Still open:** the final alias/data-flow proof that the receiver passed by the `0x17baf4` callsite is specifically the `CBluetoothApplication +0xB8` instance. That edge is not being inferred from proximity or naming.

**Evidence:** MU0678 `bluetooth` ELF; vtable `0x248208`, constructor `0x155178`, construction callsite `0x1475d8`, consumer `0x17b29c`, callers `0x17b598` and `0x17baf4`.


## 15. `enableIap` is inside the reconnect/topology decision path

The consumer trace can now be pushed beyond the raw `+0x28` read.

At `0x17b280`, the helper receives the `CBluetoothTopologyReconnect` object in `r0`, saves it in `r5`, and first refreshes/derives state through a helper call. It then reads the cached `enableIap` byte:

```text
0x17b288  MOV  r5,r0
0x17b29c  LDRB r3,[r5,#0x28]
0x17b2a4  CMP  r3,#0
```

The two branches are materially different:

```text
enableIap != 0
    -> call 0x179e40
    -> common reconnect/topology operation at 0x179618

 enableIap == 0
    -> read global FecAppMMXProxy::getIName() object
    -> read its +0x38 state field
    -> if state > 3, call 0x179618
    -> continue through 0x16d8c8
```

The global object access is not guessed from a string. The GOT slot used by the helper is independently relocated:

```text
GOT + 0x688 = 0x24cd74
R_ARM_GLOB_DAT
_ZZN3asi3fec14FecAppMMXProxy8getINameEvE5iname
```

Thus the branch is explicitly coupled to the **FEC application state** as well as `enableIap`.

There are three recovered callers of the helper:

```text
0x17b598  BL  0x17b280
0x17b938  B   0x17b280
0x17baf4  BL  0x17b280
```

The `0x17b938` caller is especially useful: it is reached after a state/update operation and passes its own `r4` as the receiver before tail-branching into the helper. The surrounding code is operating on reconnect/topology data structures, not a generic configuration object.

The binary's own diagnostic/configuration vocabulary independently confirms the subsystem boundary. The same production image contains the reconnect/topology function-name strings:

```text
checkReconnect
requestChangeTopology
updateTopology
updateReconnectInfo
isReconnectPossible
selectTopologyLogic
setTopologyLogic
```

and configuration keys:

```text
automaticReconnectDisabled
iapEnabled
bluetooth.topologyLogic
bluetooth.enableIap
bluetooth.deactivateAutomaticReconnect
```

It also contains the reconnect diagnostics:

```text
Automatic reconnect deactivated in GEM
Automatic reconnect deactivated by smartphone mode
Reconnect is suspended
Reconnect is allowed
Coding Phone allowed
```

### Interpretation boundary

**Now proven:** `bluetooth.enableIap` is not an isolated feature flag. Its cached value is consumed by code embedded in the production Bluetooth **topology/reconnect control path**, where the alternative branch also consults FEC application state.

**Still not claimed:** this does not yet prove that `enableIap` itself creates an iAP2 transport, opens an RFCOMM endpoint, or starts `btstack`. The separate `IapDeviceServices`/`CIapBTChannel` endpoint path remains a different boundary.

This is nevertheless a major architectural correction: `enableIap` belongs to the **Bluetooth topology/reconnect policy layer** that decides how Bluetooth connectivity is re-evaluated, rather than to the `+0x1182/+0x1186/+0x118A` vehicle-coding flags or to the low-level btstack process directly.

**Evidence:** MU0678 `bluetooth` ELF; `0x17b280` consumer, callers `0x17b598`, `0x17b938`, `0x17baf4`; GOT relocation `0x24cd74` for `FecAppMMXProxy::getIName()::iname`; reconnect/topology strings and configuration keys in the same production image.


## 16. Exhausted the `enableIap` consumer into the reconnect/topology machinery

The direct consumer was traced one level further through all recovered callers. The important result is that the `enableIap` read is embedded in normal Bluetooth topology/reconnect processing, not in an iAP endpoint constructor.

### 16.1 Caller at 0x17b598

The caller at `0x17b330`..`0x17b5ac` operates on a topology/reconnect data structure in `r7`. Before the `enableIap` helper it performs two other object operations and then executes:

```text
0x17b584  MOV r0,r7
0x17b588  BL  0x17aee4
0x17b58c  MOV r0,r7
0x17b590  BL  0x178ea8
0x17b594  MOV r0,r7
0x17b598  BL  0x17b280
```

The same object pointer is therefore passed through the preceding topology operations and directly into the `enableIap` consumer. There is no intervening iAP/RFCOMM endpoint API call.

### 16.2 Caller at 0x17baf4

A second caller uses a different object pointer held in `r4`:

```text
0x17bae0  MOV r1,r5
0x17bae4  BL  0x1788dc
0x17bae8  MOV r0,r4
0x17baec  BL  0x1798dc
0x17baf0  MOV r0,r4
0x17baf4  BL  0x17b280
```

Again, the helper is reached as part of object/list processing. The receiver is not produced by an iAP transport-open operation. The exact identity of this `r4` object remains a local data-flow question, but the call chain contains no concrete endpoint creation.

### 16.3 The true branch is still topology/reconnect work

The `enableIap != 0` arm at `0x17b2dc` calls `0x179e34` with:

```text
r0 = CBluetoothTopologyReconnect object
r1 = helper-local state object
```

and then joins the common operation at `0x179618`:

```text
0x17b2dc  MOV r0,r5
0x17b2e0  MOV r1,r4
0x17b2e4  BL  0x179e34
0x17b2e8  B   0x17b2c4
0x17b2c4  MOV r0,r4
0x17b2c8  BL  0x179618
```

The false arm instead consults the FEC application object through GOT slot `0x688`, checks its `+0x38` state against `3`, and only then enters the same `0x179618` operation. This is a policy/topology decision structure, not an endpoint-open structure.

### 16.4 No hidden iAP API appears in the exhausted bluetooth ELF surface

A full static name/import/string sweep of the available MU0678 `bluetooth` ELF found:

- `iapEnabled`
- `bluetooth.enableIap`
- `SERVICETYPE_IAP2`
- `ERROR_CARPLAY_ACTIVE`
- RFCOMM/service error vocabulary
- Bluetooth service/proxy registrations

but did **not** recover:

- `IapDeviceServices` in the executable's own strings/symbols;
- `CIapBTChannel`;
- `updIapDevicePath`;
- `/dev/iapDevice-`;
- the MH2p iAP2 accessory UUID;
- the iAP2 detect sequence `FF 55 02 00 EE 10`;
- a concrete RFCOMM-server creation API;
- a concrete iAP endpoint `open64()` call.

This is stronger than the earlier symbol-only negative result: the executable visibly knows about the **iAP2 service type and admission/policy state**, but the byte/string/function surface needed to prove that it itself creates the Bluetooth iAP2 resource-manager endpoint is absent.

### 16.5 Exhaustion boundary

The static trace from `bluetooth.enableIap` is therefore exhausted at the following boundary:

```text
bluetooth.enableIap
        |
        v
CBluetoothTopologyReconnect +0x28
        |
        v
0x17b280 policy helper
        |
        +--> reconnect/topology operation 0x179e34
        |        |
        |        +--> common operation 0x179618
        |
        +--> FEC-state-gated common operation 0x179618

        X--> no recovered edge to IapDeviceServices
        X--> no recovered edge to CIapBTChannel/open64
        X--> no recovered edge to RFCOMM endpoint creation
        X--> no recovered edge to btstack startup
```

This closes the currently available MU0678 `bluetooth` ELF trace. The remaining endpoint-owner question cannot be answered statically from this ELF alone. The exact MU0678 `btstack` ELF (1,912,117-byte artifact previously identified in the dump) is now the decisive missing binary for continuing this branch; alternatively, a runtime trace of the `IapDeviceServices` active-device callback containing the endpoint path would close it without btstack internals.

**Evidence:** MU0678 `bluetooth` ELF; callers `0x17b598`, `0x17b938`, `0x17baf4`; helper `0x17b280`; branches `0x17b2dc`–`0x17b2e8`; common operation `0x179618`; negative string/import sweep.
