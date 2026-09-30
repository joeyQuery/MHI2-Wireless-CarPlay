# TRACE-015 — Expanded Static Mappings: Bluetooth Policy and RFCOMM Allocation

Date: 2026-09-30

This trace closes two boundaries that TRACE-014 had left too broad:

1. MU0678 \`bluetooth.enableIap\` is now mapped from the literal configuration property through the configuration reader, object field, and policy consumer.
2. The MU0678 btstack RFCOMM channel byte in the iAP2 SDP record is now traced all the way back to the internal channel allocator. The byte is not supplied by an unidentified BlueSDK callback.

## 1. bluetooth.enableIap: exact configuration-to-object mapping

### 1.1 The literal is the actual property name

In the MU0678 \`bluetooth\` ELF:

- \`bluetooth.enableIap\` is at file offset \`0x12b2b4\`.
- With the first LOAD mapped at VA \`0x100000\`, this is VA \`0x22b2b4\`.

The CBluetoothTopologyReconnect constructor is at \`0x155178\`.

The constructor loads a GOT-relative pointer resolving to \`0x22b2b4\` and passes it through the configuration helper before storing the resulting boolean byte at object offset \`+0x28\`.

The relevant flow is:

    CBluetoothTopologyReconnect::constructor 0x155178
        |
        +-- r9 = 0x22b2b4
        |        "bluetooth.enableIap"
        |
        +-- 0x155060(r9)
        |      |
        |      +-- 0x154c3c(property-name)
        |             |
        |             +-- ConfigProvider::refresh_configuration(...)
        |             +-- PartitionManager property lookup
        |             +-- virtual value extraction at +0x64
        |
        +-- returned byte
               |
               +-- strb [this + 0x28]

This closes the identity of the configuration property feeding the cached byte. The byte is not merely adjacent to an iAP-related object; it is the value read for the literal \`bluetooth.enableIap\`.

### 1.2 The cached byte is a policy input, not btstack startup

The helper at \`0x17b280\` directly reads:

    ldrb r3, [object + 0x28]

The branch is:

    +0x28 != 0
        -> 0x179e34(...)
        -> topology/reconnect processing

    +0x28 == 0
        -> consult Bluetooth topology state
        -> continue through the alternate policy path

The helper is called from concrete callers including:

- \`0x17b598\`
- \`0x17b938\`
- \`0x17baf4\`

This means the previous conclusion needs a precise refinement:

**Resolved:** \`bluetooth.enableIap\` is the source of the cached \`+0x28\` byte and that byte gates the Bluetooth topology/reconnect policy.

**Still not proven:** that this branch directly starts btstack, creates the RFCOMM server, or creates \`/dev/iapDevice\`.

This also agrees with the independent production orchestration: btstack has its own process startup. Therefore the property should not be treated as the btstack process-start switch.

## 2. switchBluetoothAccordingToConfig is a different configuration path

\`CBluetoothApplication::switchBluetoothAccordingToConfig()\` remains at \`0x13df18\`.

Its recovered logic directly tests:

    this + 0x1182
    this + 0x1186
    this + 0x118A

and then executes Bluetooth topology/configuration actions.

The static campaign therefore separates two previously easy-to-confuse mechanisms:

    bluetooth.enableIap
        |
        v
    CBluetoothTopologyReconnect +0x28
        |
        v
    iAP permission / topology-reconnect policy

versus

    CBluetoothApplication::switchBluetoothAccordingToConfig()
        |
        v
    coding/state bytes +0x1182/+0x1186/+0x118A
        |
        v
    Bluetooth configuration switching

No evidence currently equates those three coding bytes with \`bluetooth.enableIap\`.

## 3. RFCOMM channel byte: complete producer trace

The iAP2 SDP record is at:

    VA 0x2d250c

The RFCOMM channel byte is:

    VA 0x2d2519

The IapServices registration helper is:

    0x23799c

It does:

    read IapServices + 0x0c
        |
        +--> write byte -> 0x2d2519
        |
        +--> lower SDP registration -> 0x20be48

TRACE-014 had stopped at the source member. That was premature.

### 3.1 IapServices passes the channel storage into shared allocation machinery

At \`0x2379f0\`:

    r0 = this + 0x10
    r1 = this + 0x0c
    r2 = 0
    bl  0x242574

Therefore \`this+0x0c\` is explicitly passed as writable channel storage.

### 3.2 0x242574 is a Bluetooth service-channel allocator wrapper

At \`0x242574\`:

    r0 = service object
    r1 = channel-byte pointer
    r2 = requested value/flags

The routine validates the service object and its service type, updates the service-channel state fields, and then calls:

    0x2081ac(service-object, channel-byte-pointer, flags)

Specifically it:

- reads the service's state/type at offsets around \`+0x08\` and \`+0x0c\`;
- writes the supplied mode byte into \`+0x8b\`;
- stores it as a halfword at \`+0x86\`;
- copies the service value into \`+0x8c\`;
- clears the service's \`+0x60\` field;
- calls \`0x2081ac\`.

### 3.3 0x2081ac is the actual channel allocator

At entry to \`0x2081ac\`:

    r1 -> channel byte storage

The first instruction sequence reads that byte:

    ldrb r5, [r1]

If it is already non-zero, the routine treats it as an existing channel.

If it is zero, the routine scans an internal six-entry channel table.

The allocator initializes:

    r7 = 6
    r6 = 5

and checks the corresponding internal channel records. For each free entry it writes:

    strb r7, [r1]

and returns through the normal success path.

The scan decrements through the available RFCOMM service channels.

Thus the important fact is now closed:

**The IapServices RFCOMM channel byte is allocated internally by MU0678 btstack's own channel allocator. It is not an unidentified external callback value.**

The exact first-free allocation order is the internal descending scan beginning at channel 6 and moving toward the lower available channel numbers.

### 3.4 Complete channel chain

The recovered chain is now:

    IapServices registration
          |
          v
    0x23799c
          |
          +-- read this+0x0c
          |
          +-- pass this+0x0c to 0x242574
                         |
                         v
                   0x2081ac
                         |
                         +-- inspect existing byte
                         |
                         +-- if zero:
                         |      scan internal channel table
                         |      allocate free RFCOMM channel
                         |      write byte 1..6
                         |
                         v
                 return to 0x242574
          |
          v
    0x23799c writes byte to SDP
          |
          v
    SDP record 0x2d250c
          |
          +-- RFCOMM channel byte 0x2d2519
          |
          v
    0x20be48 SDP registration

This is now a closed static producer chain.

## 4. What this says about the Bluetooth endpoint

The channel path is now independent of the endpoint pathname.

There are two distinct objects that must not be conflated:

1. the RFCOMM service channel published in the SDP record;
2. the QNX resource-manager endpoint represented by the IapDevice/resource-manager machinery.

The channel is now statically closed.

The pathname is still not equivalent to the channel byte, and the MU0678 ELF still does not prove an MH2p-style:

    /dev/iapDevice-<BT address>

name.

The exact MU0678 endpoint construction still begins from the compiled:

    /dev/iapDevice

base.

No literal MH2p suffix/publication string was found.

## 5. Updated architectural interpretation

The Bluetooth side can now be represented more precisely as:

    Bluetooth phone
          |
          v
    btstack / BlueSDK
          |
          +--> RFCOMM service registration
          |       |
          |       +--> internal channel allocator 0x2081ac
          |       |
          |       +--> iAP2 SDP record
          |
          +--> IapDevice resource-manager machinery
                  |
                  +--> /dev/iapDevice base
                  |
                  ? final pathname/address association
                  |
                  v
             IapDeviceServices
                  |
                  ? endpoint publication/serialization
                  |
                  v
               iap / CIapBTChannel

Separately:

    bluetooth.enableIap
          |
          v
    CBluetoothTopologyReconnect +0x28
          |
          v
    Bluetooth iAP permission/topology policy
          |
          ? activation of the already-compiled btstack/iAP machinery

The last arrow remains unproven.

## 6. Remaining highest-value static targets after TRACE-015

The channel-source blocker is removed.

The next static targets are now:

1. recover the exact IapDevice pathname object passed into \`resmgr_attach()\`;
2. trace the producer of that pathname object and determine whether a BT address is ever appended;
3. trace IapDeviceServices methods into the endpoint string/address returned to \`CIapBTChannel::updateiAPDevice()\`;
4. recover the actual caller/path into DIO's \`iap2_connect()\`;
5. recover the iAP2 transport-object constructor/callback table;
6. recover the \`wifi_acc_config_info()\` → AP configuration/activation boundary;
7. recover the MU0678 DIO \`interfaceName\` property writer;
8. recover the DIO \`putenv()\` → \`startMdnsdProcess()\` environment propagation;
9. recover Bluetooth/Wi-Fi identity correlation;
10. recover the CarPlay session transition after the wireless iAP2 bootstrap.

The important change is that RFCOMM channel allocation and \`enableIap\` property provenance are no longer on that unresolved list.
