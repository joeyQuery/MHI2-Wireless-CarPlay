# TRACE-007 — MU0678 iAP2 Multi-Transport Capability

**Status:** Partial

## Objective

Determine whether the MU0678 iAP2 implementation contains a transport abstraction capable of representing Bluetooth and Wi-Fi, and separate that fact from proof of an active production Wireless CarPlay transport.

## Trace Record

| Field | Current state |
|---|---|
| Entry point | ipod-drvr-iap2.so iAP2 link/Identify implementation |
| Process / binary | ipod-drvr-iap2.so |
| Caller / callee | link_create() -> transport_get_link_params(); link_send_probe() -> transport_send_pkt(); transport receive/send wrappers dispatch through transport-owned callbacks |
| Arguments | Transport object/callback details are partially recovered; exact production wireless transport object is unresolved |
| Return / error behaviour | Transport callback return handling is present; exact production wireless callback targets remain unresolved |
| IPC / ASI / DSI boundary | Not applicable inside the driver; external DIO/Bluetooth-service handoff remains unresolved |
| Device / socket / file boundary | Shipped /etc/mm/iap2.cfg selects Lightning Connector; wireless runtime endpoint remains unresolved |
| Protocol event | iAP2 Identify contains dedicated Bluetooth/Wi-Fi transport-component machinery |
| Runtime confirmation | Static binary/configuration only; no timestamped production wireless execution trace |
| Evidence IDs | E-032, E-033, E-034, E-035, E-036 |
| Remaining uncertainty | Which transport implementation is instantiated for wireless CarPlay, how it is selected, and how it reaches DIO |

## Recovered binary structure

The production ipod-drvr-iap2.so contains these transport wrappers:

~~~text
transport_get_link_params()
transport_send_pkt()
transport_receive()
transport_recv_pkt()
~~~

link_create() calls transport_get_link_params(), and link_send_probe() calls transport_send_pkt(). The wrappers dereference transport-owned callback fields, establishing a real transport abstraction rather than a USB-only packet path.

The binary also contains separate Identify handlers:

~~~text
ident_info_tspserial
ident_info_tspusbdev
ident_info_tspusbhost
ident_info_tspbt
ident_info_tspwifi
~~~

The ident_info_funcs table contains both Bluetooth and Wi-Fi handlers.

## Bluetooth transport descriptor

The static descriptor sparams_id_info_tspbt contains:

~~~text
TransportComponentIdentifier
TransportComponentName
TransportSupportsiAP2Connection
BluetoothTransportMediaAccessControlAddress
~~~

Therefore Bluetooth is represented as an iAP2 transport component with an explicit iAP2-support field and Bluetooth MAC-address field.

## Wi-Fi transport descriptor

The static descriptor sparams_id_info_wifitspcomp contains:

~~~text
TransportComponentIdentifer
TransportComponentName
TransportSupportsiAP2Connection
TransportSupportsCarPlay
~~~

This is particularly important for Wireless CarPlay: the Wi-Fi transport-component descriptor explicitly carries both iAP2-connection support and CarPlay support.

That proves compiled Wi-Fi/iAP2/CarPlay capability metadata exists in the MU0678 iAP2 driver.

It does not prove that a live CarPlay session selects this transport.

## Shipped configuration

The exact MU0678 /etc/mm/iap2.cfg contains:

~~~text
[transport]
name=Lightning Connector
id=1234
~~~

The [bluetooth] section is commented out, including its enable/id/name/connect-status fields.

Therefore the shipped configuration is USB/Lightning-oriented even though the binary contains wireless transport capability machinery.

The correct evidence classification is:

~~~text
compiled wireless iAP2 capability      = Proven
production wireless iAP2 execution     = Unproven
enableIap=true is sufficient            = Unproven
DIO accepts wireless transport directly = Unproven
~~~

## Current trace

~~~text
iAP2 driver
    |
    +--> link_create()
    |       |
    |       +--> transport_get_link_params()
    |
    +--> link_send_probe()
    |       |
    |       +--> transport_send_pkt()
    |
    +--> transport_receive()/recv_pkt()
            |
            +--> transport-owned callbacks
~~~

Identify capability:

~~~text
iAP2 Identify
    |
    +--> ident_info_tspbt
    |      +--> TransportSupportsiAP2Connection
    |      +--> BluetoothTransportMediaAccessControlAddress
    |
    +--> ident_info_tspwifi
           +--> TransportSupportsiAP2Connection
           +--> TransportSupportsCarPlay
~~~

## What this changes

The iAP2 driver must no longer be treated as a USB-only black box.

The next binary trace should recover:

1. the object passed into link_create();
2. where its transport callback table is initialized;
3. which transport implementation supplies those callbacks;
4. whether Bluetooth or Wi-Fi transport is selected in the production wireless scenario;
5. how that selected iAP2 endpoint is exposed to DIO.

## Do not infer

- sparams_id_info_tspwifi proves live Wi-Fi iAP2.
- TransportSupportsCarPlay proves a production Wireless CarPlay session exists.
- Lightning Connector means wireless code is absent.
- enableIap=true is sufficient to activate the wireless path.
- DIO's /dev/ipod0 boundary is automatically transport-neutral.

## Completion criterion

Complete only when the transport object/callback implementation is traced from construction through a production wireless connection into the DIO CarPlay session.
