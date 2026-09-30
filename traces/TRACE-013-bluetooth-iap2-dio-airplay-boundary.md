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
   +---- [unresolved bluetooth-side handoff]
   |
  DIO
   |
   +---- iAP2 / CarPlay state
   |
   +---- AirPlay / Bonjour
              |
              v
        mDNS / socket layer
              |
              v
          Wi-Fi path
```

This is an evidence-oriented convergence diagram. The dashed/unresolved handoff is not a proven execution edge.

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
- **Partial:** Bluetooth endpoint → iAP2 resource-manager/DIO handoff.
- **Partial:** DIO/AirPlay runtime interface selection.
- **Not yet proven:** end-to-end Wireless CarPlay.

## Related evidence

E-014 through E-031, E-049 through E-055.

## Related traces

- TRACE-001 — Bluetooth → iAP2
- TRACE-002 — DIO → iAP2 transport
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

## Updated remaining blocker

The remaining static question is now narrower:

> Does MU0678 btstack itself create/publish the runtime endpoint consumed through IapDeviceServices, and if so, what exact QNX resource-manager node and iAP2 ABI does it expose?

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