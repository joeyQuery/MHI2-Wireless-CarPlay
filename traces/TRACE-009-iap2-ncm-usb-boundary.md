# TRACE-009 — MU0678 iAP2-NCM / USB CarPlay Boundary

**Status:** Partial

## Objective

Determine whether the MU0678 iAP2 NCM component can be treated as the Wireless CarPlay network transport, or whether the shipped NCM implementation is explicitly tied to the existing USB CarPlay path.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | MU0678 USB CarPlay descriptor and iAP2-NCM component inventory |
| Process / binary | `devu-iap2ncm-tegra3-ci.so`; USB CarPlay descriptor; `smartphone_integrator`; `dio_manager` |
| Caller / callee | A concrete instruction-level caller chain into `devu-iap2ncm-tegra3-ci.so` is not recovered |
| Arguments | USB descriptor defines an iAP interface plus CDC-NCM control/data interfaces |
| Return / error behaviour | Not recovered for the binary itself |
| IPC / ASI / DSI boundary | USB device-mode boundary is established; binary-to-service handoff remains unresolved |
| Device / socket / file boundary | `smartphone_integrator` monitors `/dev/ipod0`; its CarPlay child is `dio_manager` |
| Protocol event | USB descriptor identifies the product as `iAP2 NCM Accessory` and exposes a CDC NCM network interface |
| Runtime confirmation | Static MU0678 configuration/descriptor evidence; no runtime Wireless CarPlay trace |
| Evidence IDs | E-044, E-037, E-005 |
| Remaining uncertainty | Exact callers/loader of `devu-iap2ncm-tegra3-ci.so`, and whether any separate wireless path reuses the same NCM implementation |

## Recovered USB descriptor

The MU0678 dump contains:

```text
product = 'iAP2 NCM Accessory'
```

The descriptor exposes:

```text
1. vendor-specific iAP Interface
2. CDC Communications / NCM control interface
3. CDC Data interface
4. CDC Data alternate setting with bulk IN/OUT
```

It also contains the CDC Ethernet Networking Functional Descriptor and CDC NCM Functional Descriptor.

Therefore the shipped MU0678 image has an explicit USB device-mode product definition combining iAP2 with NCM networking.

This is materially stronger than merely observing the filename `devu-iap2ncm-tegra3-ci.so`.

## Correlation with production CarPlay orchestration

The production `smartphone_integrator.json` contains:

```text
paths.mcdMonitored = ["/dev/ipod0"]
```

and defines the CarPlay child as:

```text
exec = "dio_manager"
path = "/mnt/app/eso/bin/apps"
```

This establishes the shipped smartphone orchestration boundary as USB-device-driven.

Separately, the production iAP2 configuration selects:

```text
[transport]
name=Lightning Connector
id=1234
```

while its Bluetooth section is commented out.

Taken together, these findings provide MU0678-specific evidence that the recovered NCM/iAP2 configuration belongs to the existing wired/USB CarPlay architecture.

## What this does NOT prove

It does **not** prove that:

- `devu-iap2ncm-tegra3-ci.so` can never be reused by a wireless transport;
- the binary has no wireless-capable code;
- the Bluetooth iAP endpoint cannot eventually feed the same transport-neutral iAP2 service;
- Wireless CarPlay is impossible on this hardware;
- changing the NCM configuration would enable Wireless CarPlay.

Those questions require the binary-level caller/loader and transport-object trace.

## Current conclusion

The previous hypothesis:

```text
iAP2 NCM component
        ↓
possibly Wireless CarPlay transport
```

must **not** be promoted to fact.

The stronger evidence-backed model is:

```text
USB CarPlay
   ↓
USB device-mode descriptor
   ├── iAP Interface
   └── CDC NCM network interface
          ↓
      /dev/ipod0 / CarPlay orchestration
          ↓
      smartphone_integrator
          ↓
      dio_manager
```

The Bluetooth path remains separate and unresolved at the transport-selection boundary:

```text
Bluetooth
   ↓
IapDeviceServices
   ↓
CIapBTChannel
   ↓
runtime endpoint
   ↓
??? iAP2 transport
```

The next trace must therefore focus on the concrete Bluetooth endpoint owner and the transport object selected after `open64()`, rather than assuming the USB NCM component is the wireless bridge.

## Completion criterion

Complete this trace only when the actual caller/loader and transport-object relationship for `devu-iap2ncm-tegra3-ci.so` is recovered from MU0678 binary evidence.
