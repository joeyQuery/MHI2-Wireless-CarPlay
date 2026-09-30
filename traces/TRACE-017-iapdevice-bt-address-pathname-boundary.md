# TRACE-017 — IapDevice Bluetooth-address fields and pathname provenance boundary

Date: 2026-09-30

This trace exercises the next exact endpoint lead after TRACE-016. It closes one previously ambiguous IapDevice field mapping and narrows the remaining pathname question further.

## 1. The IapDevice +0x28/+0x2c fields are the Bluetooth address

The non-virtual IapDevice setter is at `0x23e394`.

Its recovered write path is:

```
0x23e394
    r0 = IapDevice
    r1 = address/associated string-state object
    r2 = low 32 bits of address
    r3 = high 32 bits of address
        |
        +--> [IapDevice + 0x28] = r2
        +--> [IapDevice + 0x2c] = r3
```

The caller at `0x23cd8c` supplies an actual six-byte Bluetooth address.

Immediately before the call:

- `r3 = [device-object + 0x0c]`;
- bytes at offsets `+0x0d` through `+0x12` are read;
- those six bytes are assembled into two 32-bit words in `r6:r7`;
- `r2 = r6`;
- `r3 = r7`;
- `BL 0x23e394`.

The reconstruction is:

```
device object + 0x0d..0x12   (6 bytes)
            |
            v
       r6:r7 = 48-bit address
            |
            v
0x23cd8c -> 0x23e394
            |
            +--> IapDevice +0x28
            +--> IapDevice +0x2c
```

The other recovered caller at `0x23c36c` passes zero/zero to the same setter, which is consistent with clearing the stored address.

Therefore the earlier description of `+0x28/+0x2c` as an unspecified identifier/value is corrected:

**MU0678 btstack IapDevice +0x28/+0x2c are the stored 48-bit Bluetooth device address.**

## 2. This does not prove the address is used to name the endpoint

The pathname path remains separate.

The exact pathname consumed by QNX is:

```
IapDevice +0x0c
       |
       v
resmgr_attach() @ 0x23fc84
```

The creation path immediately before construction is:

```
sp+0x154 = source pointer
sp+0x150 = source length
        |
        +--> malloc(length + 1)
        +--> memcpy(new_buffer, source, length)
        +--> NUL terminate
        |
        v
sp+0x140 / sp+0x144 / sp+0x148
        |
        v
IapDevice constructor @ 0x23fe24
        |
        v
IapDevice +0x0c
        |
        v
resmgr_attach @ 0x23fc84
```

The BT address fields at `+0x28/+0x2c` are written by `0x23e394), but no recovered instruction in the inspected pathname-construction path formats those fields into the pathname.

In particular, the MU0678 ELF still contains no literal:

```
/dev/iapDevice-
```

and no recovered `snprintf`/format operation in this path establishes an address-derived suffix.

Therefore the MH2p convention:

```
/dev/iapDevice-<16-hex-BT-address>
```

remains reference-only.

## 3. The remaining pathname source pair has an anomalous stack provenance

The sole recovered direct caller of the large IapDevice-creation case at `0x23c028` is `cbRf` at `0x23d500`.

The call is:

```
cbRf @ 0x23d500
    |
    +--> r0 = r5
    +--> r1 = r5
    +--> r2 = r6
    |
    +--> BL 0x23c028
```

The callee later reads:

```
[sp + 0x154]  -> source pointer
[sp + 0x150]  -> source length
```

and there are no direct stores to these two stack slots in the recovered body before their first use.

The important consequence is negative:

**The source pointer/length pair is not locally produced by the visible IapDevice-creation code.**

The currently accessible binary does not justify inventing a local producer for those slots.

The next possible producer is therefore at the callback/ABI boundary represented by `cbRf` and the indirect BlueSDK/RFCOMM machinery that invokes it, or in stack state outside the recovered function body.

## 4. cbRf is itself a real production callback boundary

`cbRf` is not an invented helper. It has a defined address and a dynamic relocation:

```
R_ARM_GLOB_DAT
GOT 0x2d18b0
symbol: cbRf
target: 0x23d500
```

The wrapper checks two incoming objects and, when the relevant state is present, calls `0x23c028`.

The exact callback registration ABI that supplies the additional pathname source state is not recovered statically from the accessible btstack ELF.

This is now the precise endpoint-path stopping boundary.

## 5. Closed endpoint facts after TRACE-017

The MU0678 endpoint architecture is now:

```
Bluetooth device address
        |
        +--> 0x23cd8c
        |
        v
IapDevice +0x28/+0x2c
        |
        |   [address storage]
        |
        X--> no recovered static edge to pathname formatting
       
runtime pathname source pointer/length
        |
        v
IapDevice +0x0c
        |
        v
resmgr_attach @ 0x23fc84

compiled /dev/iapDevice
        |
        v
surrounding endpoint-construction state
        |
        X--> final equality to +0x0c pathname not proven
```

This separates three facts that must not be conflated:

1. MU0678 stores the connected Bluetooth MAC in the IapDevice object.
2. MU0678 passes a separately prepared runtime pathname to `resmgr_attach`.
3. The available static evidence does not prove that the MAC is formatted into that pathname.

## 6. Exhaustion boundary

The exact remaining static question is now:

**What code/ABI supplies `sp+0x154` (pathname source pointer) and `sp+0x150` (pathname length) before the IapDevice construction case at `0x23c028)?**

The accessible MU0678 btstack ELF does not expose a direct local store to those slots, and the visible caller `cbRf` does not populate them through ordinary register arguments.

This should not be replaced by the MH2p naming convention without either:

- a recovered BlueSDK callback-registration/ABI path that supplies the values, or
- a runtime capture of the actual `resmgr_attach` pathname.

## Evidence

- E-092 — IapDevice +0x0c is the exact pathname passed to `resmgr_attach`.
- E-093 — pathname is copied from a runtime source pointer/length immediately before construction.
- E-094 — compiled `/dev/iapDevice` is not proven equal to the final pathname.
- New E-095 — IapDevice +0x28/+0x2c are the stored Bluetooth device address, proven by the `0x23cd8c -> 0x23e394` data flow.
- New E-096 — pathname source slots `sp+0x154/sp+0x150` have no recovered local producer in the creation case; `cbRf` is the sole direct caller and forms the remaining callback/ABI boundary.
