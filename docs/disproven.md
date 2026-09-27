# Disproven / Eliminated Interpretations

Permanent record of interpretations that investigation has eliminated, corrected or shown unsafe to assume. This prevents repeated dead-end investigations.

## 1. `hci0` is not proven to be the production Bluetooth transport

The image contains a lab/manufacturing HCI configuration involving `hci0`, `/dev/ttyS0`, TCP and `mlan0`. Production architecture uses the Marvell 8787 SDIO platform. The lab configuration cannot establish the production HCI transport.

**Status:** Eliminated as a production assumption.

**Evidence:** E-012.

## 2. `\/dev\/ttyS0` is not proven to be production Bluetooth HCI

Same production/lab evidence boundary as `hci0`.

**Status:** Eliminated as a production assumption.

## 3. `uap0` is not yet proven to replace `carplay0`

The existence of `uap0` and AirPlay Wi-Fi-aware helpers does not prove that changing the CarPlay network interface from `carplay0` to `uap0` is sufficient.

**Status:** Unproven; do not treat as a solved substitution.

**Trace:** TRACE-003.

## 4. `MDNS_DIRECTLINK_IFACE=uap0` is not yet proven sufficient

The production value `carplay0` is known, but the consumer and complete runtime socket/interface path have not been recovered.

**Status:** Unproven.

**Trace:** TRACE-003.

## 5. Presence of iAP2 binaries does not prove Wireless CarPlay iAP2 is enabled

The image contains multiple iAP/iAP2 components, including a Bluetooth iAP proxy, but their presence alone does not establish a working Bluetooth CarPlay bootstrap.

**Status:** Eliminated as an inference.

**Evidence:** E-007.

## 6. Lab HCI evidence must not substitute for production HCI evidence

Manufacturing/lab configuration is useful evidence about available interfaces, but it cannot establish the production HCI device or transport without a production-specific trace.

**Status:** Methodological limitation established.

## 7. AirPlay Wi-Fi helper presence does not prove MHI2 invokes AirPlay on `uap0`

The existence of `IsWiFiNetworkInterface` and interface-selection functions establishes capability in the library, not the runtime argument values supplied by DIO.

**Status:** Unproven; preserved as an open trace.

**Trace:** TRACE-004 and TRACE-005.

## 8. Component presence does not prove runtime participation

A binary, library, configuration key or symbol being present establishes availability in the image, not participation in a specific production path.

**Status:** Established project-wide evidence rule.

## 9. AirPlay screen setter APIs are not the active production transport selector

The production `libairplay.so` exports `AirPlayReceiverSessionScreen_SetIFName`, `AirPlayReceiverSessionScreen_SetTransportType` and `AirPlayReceiverSessionScreen_SetClientIfMACAddr`, but the implementations reduce to `bx lr`.

**Status:** Disproven as the presumed active selector for this production build.

**Evidence:** E-027; TRACE-004.

## 10. `SocketSetBoundInterface` is not the active production binding mechanism

The production entry is a tiny stub/constant-return. The substantive network helpers that remain relevant are `SocketSetPacketReceiveInterface` and `SocketSetMulticastInterface`.

**Status:** Disproven as the presumed production binding mechanism.

**Evidence:** E-019, E-020; TRACE-005.

## 11. Bluetooth iAP endpoint is not proven to be `/dev/ipod0`

DIO independently proves `/dev/ipod0` through its `iap2.device` configuration and `iap2_connect` callsites. The Bluetooth `iap` binary independently proves a runtime-supplied endpoint passed to `open64()`, and contains no `/dev/ipod0` literal.

**Status:** Endpoint equality remains unproven; treating them as identical is eliminated as an assumption.

**Evidence:** E-016, E-021, E-022, E-023; TRACE-001 and TRACE-002.

## Recording New Eliminations

When an investigation disproves an interpretation:

1. record the old interpretation;
2. record the concrete evidence;
3. state the corrected interpretation;
4. link the relevant evidence-register ID;
5. update affected subsystem documents so the old claim cannot silently survive elsewhere.
