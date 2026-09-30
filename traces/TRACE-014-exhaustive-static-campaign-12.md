# TRACE-014 — Exhaustive Static Campaign: 12 Remaining Wireless CarPlay Traces

Date: 2026-09-30

This trace records the result of a deliberate static-only pass across all twelve remaining implementation targets. It does not convert unresolved runtime questions into assumptions. Where the accessible MU0678 artifacts do not contain the next edge, the exact boundary is recorded.

## 1. iAP2 transport object / callback construction

**Result: partially traced; static boundary reached.**

Proven in MU0678 ipod-drvr-iap2.so:

- link_create() reaches transport_get_link_params().
- link_send_probe() reaches transport_send_pkt().
- send/receive wrappers dispatch through transport-owned callback fields.
- ident_info_tspbt() and ident_info_tspwifi() are separate transport-component Identify handlers.
- The Wi-Fi transport descriptor advertises TransportSupportsiAP2Connection and TransportSupportsCarPlay.
- The Bluetooth descriptor advertises TransportSupportsiAP2Connection and BluetoothTransportMediaAccessControlAddress.

The remaining static edge is the constructor/initializer that supplies the concrete transport callback table to the object passed to link_create(), followed by production selection of Bluetooth versus Wi-Fi. The currently committed binary evidence does not recover that producer.

## 2. iAP2 Wi-Fi configuration handler → connectionmanager / uap0

**Result: control-plane capability proven; cross-process producer/consumer edge unresolved.**

link_handle_iap2pkt() reaches:

- 0x13818 -> bt_update_recv()
- 0x1382c -> wifi_acc_config_info()

The latter handles RequestAccessoryWiFiConfigurationInformation and contains the SSID/passphrase/security/channel/WPA2 response machinery.

MU0678 also has a separate connectionmanager AP implementation and uap0, but the available static evidence does not establish a direct call/IPC edge from wifi_acc_config_info() into connectionmanager, nor that this handler is the production wireless-CarPlay AP activation path.

## 3. DIO notifyiAP2DeviceConnected → iAP2Connect → requestCarPlay

**Result: DIO iAP2 ABI path closed to the caller-supplied endpoint; session transition remains unresolved.**

The production DIO side establishes:

CIpodAP2Service vtable target 0x15e240
→ helper 0x15ddb4
→ imported iap2_connect at 0x115de8.

At 0x15e2f4, a runtime path pointer is supplied to the helper. The current production configuration supplies /dev/ipod0.

Separately, libiap2client.so::iap2_connect() passes its first argument directly to open() and then sends the 20-byte QNX iAP2 control message. The driver checks the same 0x9999 marker at message offset +0x06.

The unresolved edge is which caller supplies the path for a wireless session and how the resulting iAP2 connection drives requestCarPlay() / checkCarPlayCompatibility() / session creation.

## 4. btstack IapDeviceServices publication / serialization

**Result: service infrastructure substantially traced; exact endpoint publication remains open.**

MU0678 btstack contains:

- IapDeviceServicesServiceRegistration
- IapDeviceServicesS
- IapDeviceServicesProxyReply
- interface asi.connectivity.bluetooth.iap.IapDeviceServices
- IapServices RFComm/iAP-SDP registration strings and methods.

The SDP record at 0x2d250c contains L2CAP, RFCOMM UUID 0x0003, a runtime channel byte, and the iAP2 accessory UUID 00000000-deca-fade-deca-deafdecacaff. The record is directly associated with the IapDeviceServices registration name.

The remaining edge is the service method that serializes the connected device endpoint/address/channel into the proxy reply consumed by the production iap process.

## 5. /dev/iapDevice suffix construction

**Result: endpoint owner and base construction proven; suffix not proven.**

The exact MU0678 btstack ELF contains:

- btstack::IapDevice vtable at 0x2d0a58;
- endpoint-construction path beginning around 0x23c720;
- exact string /dev/iapDevice at file offset 0x1bf790 / VA 0x2bf790;
- strlen() consumption of that string;
- iPL string-state construction;
- resource-manager construction at 0x23fe24;
- resmgr_attach() at 0x23fc84.

A direct byte search of the exact ELF found no literal /dev/iapDevice-, updIapDevicePath, Creating iapDevice, or Received correct iAP2 handshake.

That is useful negative evidence. It means the MH2p /dev/iapDevice-<BT address> convention cannot be copied into MU0678 as though it were already proven. A suffix could still be assembled through non-literal C++ operations, but no such suffix-producing edge was recovered in the inspected endpoint-construction path.

## 6. IapServices +0x0c RFCOMM channel producer

**Result: traced to the shared machinery boundary.**

0x23799c reads IapServices + 0x0c and writes byte 0 into the SDP RFCOMM channel byte at 0x2d2519, then reaches local SDP routine 0x20be48.

The source byte is not a standalone proven channel variable. The IapServices constructor zeroes the embedded region beginning at +0x0c.

Further tracing shows:

IapServices +0x10 and IapServices +0x0c
→ shared 0x242574
→ byte 0 of the +0x0c structure
→ lower-level 0x2081ac.

Other callers at 0x2563e0 and 0x25dbc0 establish that 0x242574 is shared machinery.

No direct IapServices byte-0 writer was recovered. This is the current static boundary.

## 7. DIO AirPlay interfaceName population

**Result: AirPlay consumer is closed; MU0678 DIO writer remains unresolved.**

Production libairplay.so uses:

object + 0x6c interfaceName
→ if_nametoindex()
→ DNSServiceRegister(... interfaceIndex, ...).

The screen IFName/transport/client-MAC setter exports are no-op stubs.

MU0678 dio_manager has a concrete AirPlayReceiverServerCreate() call at 0x13f0b4, AirPlayReceiverServerSetDelegate() at 0x13f34c, and an R_ARM_GLOB_DAT relocation for AirPlayReceiverServerSetProperty at 0x19da70. The GLOB_DAT linkage is real, but a direct property callsite has not been recovered.

CFObjectSetPropertyCString is imported and called elsewhere, but no static proof currently ties one of those calls to the AirPlay server's interfaceName property.

The MH2p reference proves a similar pattern on a different firmware family; it is reference evidence only, not MU0678 evidence.

## 8. AirPlay packet/multicast socket helper callers

**Result: negative DIO-level boundary established; callers elsewhere remain unresolved.**

MU0678 dio_manager does not import:

- SocketSetPacketReceiveInterface
- SocketSetMulticastInterface
- SocketSetBoundInterface
- IsWiFiNetworkInterface.

Therefore these are not DIO-level direct calls.

libairplay.so still contains substantive packet/multicast and Wi-Fi-classification helpers, while SocketSetBoundInterface is a stub/constant-return. The production caller chain into the substantive helpers is not recovered from the currently accessible artifacts.

## 9. DIO MDNS_DIRECTLINK_IFACE producer → startMdnsdProcess

**Result: capability and consumer proven; producer propagation unresolved.**

MU0678 contains:

- MDNS_DIRECTLINK_IFACE=carplay0
- setEnv
- getEnvName
- putenv(%s): %s
- startMdnsdProcess
- restartMdnsd
- stopMdnsdProcess.

mdnsd independently consumes MDNS_DIRECTLINK_IFACE with getenv() in SetupOneInterface().

The exact instruction-level chain from DIO environment-setting code to the process environment of the running mdnsd has not been recovered. The production preparation script also does not export the variable.

The MH2p boot script and MU1329 DIO-spawn model demonstrate two reference designs, but neither is proof of MU0678 propagation.

## 10. BluetoothSmartphoneIntegration → CarPlay state

**Result: Bluetooth policy/state boundary proven; wireless-CarPlay handoff unresolved.**

MU0678 bluetooth contains concrete CBluetoothSmartphoneIntegration and topology/reconnect machinery, including:

- iapEnabled
- bluetooth.enableIap
- SERVICETYPE_IAP2
- ERROR_CARPLAY_ACTIVE
- Carplay
- requestChangeTopology
- topology update/reconnect methods.

switchBluetoothAccordingToConfig() at 0x13df18 is called from diagCBCodingValues() at 0x146500 and checks state bytes at this+0x1182, +0x1186, and +0x118A.

No static proof maps those bytes directly to bluetooth.enableIap, and no direct endpoint-construction/RFCOMM-server API was recovered from the Bluetooth executable.

Thus this layer proves policy/state integration, not wireless iAP2 activation.

## 11. ipod-drvr-iap2 resource-manager mountpoint construction

**Result: driver mount operation closed; runtime mountpoint producer unresolved.**

The driver object is proven:

- ipod_module at 0x27e7c;
- iap2_drvr at 0xa9bb0;
- resource-manager handlers include iap2_init, iap2_msg, iap2_notify, iap2_unblock, OCB lifecycle handlers and filesystem operations.

During iAP2 feature startup, iap2_start_features() calls ipod_resmgr_mount(r9, 1) where r9 is loaded from the iAP2 context at +0x14.

Together with the exact DIO iap2_connect(path, flags) ABI, this proves the client boundary is structurally path-driven.

The unresolved static question is the producer of context +0x14 and whether that mountpoint is the Bluetooth btstack::IapDevice service.

## 12. Boot / configuration dependency graph

**Result: production component graph substantially closed; wireless activation graph remains partial.**

MU0678 production orchestration proves independent startup of the major components:

- btstack
- bluetooth
- connectionmanager
- dio_manager
- iAP2 components
- WLAN/AP infrastructure.

The AP side has uap0 and the iAP2 driver has executable Wi-Fi configuration machinery. DIO owns the current /dev/ipod0 CarPlay iAP2 path and creates the AirPlay server. Bluetooth owns a separate iAP endpoint/service stack.

The key static dependency graph is:

    bluetooth / btstack
            |
            +--> IapDeviceServices / IapServices
            |          |
            |          +--> RFCOMM SDP registration
            |          |
            |          +--> /dev/iapDevice resource-manager endpoint
            |
            +--> Bluetooth phone identity / topology state
                       |
                       ? wireless iAP2 bootstrap
                       |
                       v
              ipod-drvr-iap2 transport layer
                       |
                       ? endpoint / mountpoint convergence
                       |
                       v
                 DIO iAP2 boundary
              iap2_connect(runtime path)
                       |
                       ? requestCarPlay / compatibility
                       |
                       v
                 AirPlay / Bonjour
                       |
                       ? interfaceName writer
                       |
                       v
                      uap0

    connectionmanager --> uap0 / AP lifecycle
    ipod-drvr-iap2     --> Wi-Fi config response machinery
    mdnsd              --> MDNS_DIRECTLINK_IFACE consumer

Every ? is still a real missing MHI2 edge. None has been promoted from reference firmware evidence.

## Static campaign conclusion

All twelve requested targets were re-audited against the available MU0678 binaries, configuration evidence, and the repository's existing disassembly traces.

The highest-value remaining static boundaries are now narrowly defined:

1. concrete transport-object/callback initializer in ipod-drvr-iap2.so;
2. wireless iAP2 Wi-Fi configuration → AP activation handoff;
3. wireless caller/path into DIO iap2_connect();
4. IapDeviceServices endpoint serialization;
5. BT-address → final /dev/iapDevice... pathname;
6. producer of IapServices +0x0c byte through shared 0x242574/0x2081ac;
7. MU0678 DIO AirPlay interfaceName property writer;
8. AirPlay helper callers outside DIO;
9. exact putenv/mdnsd process-environment propagation;
10. Bluetooth topology/iAP policy → wireless activation;
11. ipod_resmgr_mount() mountpoint producer and convergence with btstack;
12. the complete boot-to-session dependency graph.

Several of these can only be closed by additional firmware artifacts or runtime traces. The repository now records the exact static stopping points instead of treating them as broad unknowns.
