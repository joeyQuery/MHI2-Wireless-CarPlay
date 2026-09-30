# TRACE-018 — IapDevice callback/pathname boundary and IapDeviceServices reply surface

Date: 2026-09-30

Continues TRACE-017 using the exact MU0678 btstack ELF.

## 1. cbRf -> IapDevice construction

cbRf @ 0x23d500 takes two object inputs. When both are non-null and [A+0x14] is non-zero it calls 0x23c028 with r0=A, r1=A, r2=B. It does not pass an endpoint pathname in the normal argument registers.

At 0x23c094, 0x23c028 loads the event/device object from sp+0x20 and dispatches on its first byte:

event byte - 1: 0 -> 0x23c0c4; 1 -> 0x23c720; 2 -> 0x23c318; 3 -> 0x23c42c; 4 -> 0x23c1f8; 5 -> 0x23c1b4.

The 0x23c720 branch is therefore an event-specific IapDevice creation case.

## 2. Bluetooth MAC and pathname are separate state

Before the event dispatch, 0x23c0d4 loads event_object+0x0c and reads six bytes at +0x0d..+0x12, reconstructing the 48-bit Bluetooth address in r6:r7. That value reaches setter 0x23e394, which stores it at IapDevice+0x28/+0x2c.

The pathname construction is separate. The 0x23c720 case resolves compiled /dev/iapDevice at 0x2bf790 and calls strlen(), but the final IapDevice pathname state is built from sp+0x154/sp+0x150:

0x23c7c0: load sp+0x154
0x23c7d4: load sp+0x150
0x23c7dc: construct the pathname string state
0x23c884: call IapDevice constructor 0x23fe24
0x23fc84: resmgr_attach(pathname = IapDevice+0x0c)

No direct store to sp+0x154 or sp+0x150 exists in the inspected 0x23c028 dispatcher body. Those values are pre-existing state at the creation case.

Therefore MU0678 still does not statically prove /dev/iapDevice-<BT address>, and the compiled /dev/iapDevice string cannot be equated with the final resmgr_attach pathname.

## 3. IapDeviceServices reply surface

Exact MU0678 btstack objects:

- IapDeviceServicesServiceRegistration vtable 0x2d0c80
- IapDeviceServicesProxyReply vtable 0x2d0d08
- IapDeviceServicesS vtable 0x2d0d50
- interface name asi.connectivity.bluetooth.iap.IapDeviceServices at 0x2bf7a0

Adjacent service strings are: updateActiveDevice, unregisterSelf, clientConnected, updateLocalBtAddress, clientDisconnected, stringRequest, IapMediaBridge, updateLocalFriendlyName.

The binary also contains concrete diagnostics for reply->updateLocalBtAddress(address), reply->updateLocalFriendlyName(name), and reply->updateActiveDevices(array). This proves production reply machinery for local Bluetooth address, friendly name, and active-device state.

The exact generated IPC serialization layout that carries the endpoint pathname into the remote proxy is not exposed as a unique exported symbol in the stripped image.

## 4. Closed separation

Bluetooth device -> MAC -> IapDevice +0x28/+0x2c
Bluetooth device -> IapServices -> RFCOMM channel -> SDP 0x2d2519
Bluetooth device -> IapDeviceServices -> active-device/address reply API
IapDevice -> runtime pathname -> resmgr_attach 0x23fc84

These are distinct static edges. No evidence currently proves which IapDeviceServices message supplies the final QNX pathname to CIapBTChannel.

## Evidence

E-095 — IapDevice +0x28/+0x2c store the connected Bluetooth address.
E-096 — pathname source pair remains outside the recovered local producer chain.
E-097 — cbRf dispatches into the multi-event IapDevice construction case with event/device objects, not an endpoint pathname argument.
E-098 — IapDeviceServices exposes production active-device/local-address/friendly-name reply machinery in the exact MU0678 btstack ELF.