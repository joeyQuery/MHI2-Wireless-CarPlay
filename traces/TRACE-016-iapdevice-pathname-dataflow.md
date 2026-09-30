# TRACE-016 — IapDevice pathname data-flow and resmgr_attach boundary

Date: 2026-09-30

This trace tightens the endpoint mapping and corrects an over-broad statement in E-085.

## 1. The pathname passed to QNX is IapDevice +0x0c

The IapDevice constructor is reached at 0x23fe24.

Its first meaningful string operation loads the character pointer from [r1+0x08] and the length/state from [r1+0x04], then stores the resulting string pointer into this+0x0c.

The resource-manager attach path at 0x23fc84 then loads [IapDevice+0x0c] into the pathname argument register and calls resmgr_attach.

Therefore:

    IapDevice +0x0c
          |
          v
    resmgr_attach pathname argument

is closed.

## 2. The pathname is copied from a runtime string

The sole recovered constructor caller is the btstack path at 0x23c884.

Immediately before the constructor:

    sp+0x154 = source character pointer
    sp+0x150 = source length

The code loads the length, allocates length+1, copies the bytes from sp+0x154, and writes a terminating NUL.

It then constructs the string-state object:

    sp+0x140 = length
    sp+0x144 = length
    sp+0x148 = new_buffer

and passes r1 = sp+0x140 to 0x23fe24.

The constructor consequently stores [sp+0x148] into IapDevice+0x0c, and resmgr_attach consumes that pointer.

## 3. What /dev/iapDevice actually does

At 0x23c720 the code resolves the single /dev/iapDevice literal at VA 0x2bf790, calls strlen(), and constructs an independent ipl basic-string State around that base string.

That string state is stored inside the newly allocated btstack object before the IapDevice constructor is called.

However, the actual pathname argument to IapDevice is the separately constructed string state at sp+0x140, whose source bytes are sp+0x154.

Correct static conclusion:

- /dev/iapDevice is compiled into MU0678 btstack.
- It participates in the IapDevice creation path.
- IapDevice has a concrete pathname field at +0x0c.
- +0x0c is passed directly to resmgr_attach().
- The pathname is copied from a runtime string source immediately before construction.
- It is not yet proven that the final pathname equals exactly /dev/iapDevice.
- It is not proven to be /dev/iapDevice-<BT address>.
- No BT-address append operation has been recovered on the final pathname path.

## 4. Endpoint mapping now has a precise remaining boundary

    runtime source string
          |
          | source pointer + length
          v
    malloc + memcpy
          |
          v
    temporary basic-string state
          |
          v
    IapDevice constructor 0x23fe24
          |
          v
    IapDevice +0x0c
          |
          v
    resmgr_attach 0x23fc84
          |
          v
    QNX /dev pathname

The unresolved producer is specifically the source pair:

    sp+0x154 = pathname bytes
    sp+0x150 = pathname length

## 5. BT-address question

The exact MU0678 btstack binary contains no literal /dev/iapDevice- and no updIapDevicePath.

The recovered final pathname data-flow also does not expose a BT-address formatting operation.

Therefore the MH2p /dev/iapDevice-<BT address> convention remains reference evidence only.

## 6. Object-field separation

The IapDevice constructor stores independent fields including:

    +0x0c pathname
    +0x10 allocation/container pointer
    +0x14 allocation/container state
    +0x18 constructor-supplied value
    +0x1c constructor-supplied value
    +0x20 constructor-supplied value
    +0x24 resmgr_attach return value
    +0x28 later-set identifier/value
    +0x2c later-set identifier/value

The later setter at 0x23e394 writes supplied r2/r3 into +0x28/+0x2c. Those fields are therefore separate from the pathname.

## 7. Next exact target

Trace the producer of sp+0x154/sp+0x150 backward through the IapDevice creation case and, if necessary, through the Bluetooth service/device object supplying the string.
