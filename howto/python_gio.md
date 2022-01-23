# D-Bus and Bluez

## Introduction <a name="introduction"></a>

If you want to do Bluetooth on Linux then BlueZ (the official Bluetooth stack
on Linux) is the best option. They have made a number of APIs available for
people to interface with their functionality enabling people to write
their own applications. 
These BlueZ APIs use D-Bus which is not widely known about so this article
aims to give some background to help people get started with writing 
applications using BlueZ's D-Bus APIs.

## Table of contents

1. [Introduction](#introduction)
2. [D-Bus](#dbus)
    1. [Buses](#buses)
    2. [Bus Name](#bus_name)
    3. [Object Path](#object_path)
    4. [Interface](#interface)
3. [D-Bus Bindings For Python](#bindings)
4. [Creating a proxy for a BlueZ object](#proxy)
5. [Discovering BlueZ Managed Objects](#managed_objects)
6. [Getting Properties](#properties)
7. [Asynchronous Event Loop](#async)
8. [D-Bus Properties Changed](#properties_changed)
9. [Building a BLE Central](#ble_central)


## D-Bus <a name="dbus"></a>

D-Bus allows communication between multiple processes running concurrently 
on the same machine. In the case of BlueZ this is between the Bluetooth
Daemon (`bluetoothd`) and the application you have written.

An advantage of D-Bus is that most programming languages have bindings to it
so the BlueZ APIs are language agnostic. D-Bus bindings require you to know
four bits of information:
1. Which bus (`system` or `session`) the service is on
2. The `bus name` of the service
3. The `object path` of the methods and properties
4. The `interface` for the `method`, `property` or `signal` 

This hierarchy of information builds to uniquely identify what is to be
accessed with D-Bus.

    System Bus -> Bus Name -> Object Path -> Interface -> Method

Let's look at each of these in more detail.

### Buses <a name="buses"></a>

There are two buses:
 - a `session` bus for each user login session
 - a single `system` bus that provides access to system services

For BlueZ and accessing `bluetoothd` we will always use the `system` bus

### Bus Name <a name="bus_name"></a>

To see all the `system` D-Bus names available on your machine run the following
command line:
```shell
$ busctl list
```
In the list will be `org.bluez` and this is what is used for BlueZ 
`bluetoothd` information.

### Object Path <a name="object_path"></a>

The `object path` looks like a filesystem path, but they are not, they are 
identifiers of unique objects on the D-Bus.
For example the default Bluetooth adapter on a Linux system is typically
`/org/bluez/hci0`
A list of all the objects being managed by BlueZ can be got on the command line
with the following command:
```text
$ busctl tree org.bluez
└─/org
  └─/org/bluez
    └─/org/bluez/hci0
      └─/org/bluez/hci0/dev_12_34_56_78_9A_BC
        └─/org/bluez/hci0/dev_12_34_56_78_9A_BC/service000a
          └─/org/bluez/hci0/dev_12_34_56_78_9A_BC/service000a/char000b
            └─/org/bluez/hci0/dev_12_34_56_78_9A_BC/service000a/char000b/desc000d
```

### Interface <a name="interface"></a>

Think of an interface as a named group of methods and signals. D-Bus identifies
interfaces with a simple namespaced string.
There may be a number of interfaces on an object path. For example on 
`/org/bluez/hci0` there can be:

    org.bluez.Adapter1
    org.bluez.GattManager1
    org.bluez.LEAdvertisingManager1
    org.bluez.Media1
    org.bluez.NetworkServer1
    org.freedesktop.DBus.Introspectable
    org.freedesktop.DBus.Properties

We can find out what is on the object by introspection. Here is an example:
```text
$ busctl introspect org.bluez /org/bluez/hci0
NAME                                TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.bluez.Adapter1                  interface -         -                                        -
.GetDiscoveryFilters                method    -         as                                       -
.RemoveDevice                       method    o         -                                        -
.SetDiscoveryFilter                 method    a{sv}     -                                        -
.StartDiscovery                     method    -         -                                        -
.StopDiscovery                      method    -         -                                        -
.Address                            property  s         "FE:FB:AC:8F:0C:A4"                      emits-change
.AddressType                        property  s         "public"                                 emits-change
.Alias                              property  s         "LinuxMachine"                             emits-change writable
.Class                              property  u         786700                                   emits-change
.Discoverable                       property  b         false                                    emits-change writable
.DiscoverableTimeout                property  u         190                                      emits-change writable
.Discovering                        property  b         false                                    emits-change
.Modalias                           property  s         "usb:v1D6Bp0246d0535"                    emits-change
.Name                               property  s         "LinuxMachine"                             emits-change
.Pairable                           property  b         false                                    emits-change writable
.PairableTimeout                    property  u         120                                      emits-change writable
.Powered                            property  b         true                                     emits-change writable
.UUIDs                              property  as        9 "0000110e-0000-1000-8000-00805f9b34fb… emits-change
org.bluez.GattManager1              interface -         -                                        -
.RegisterApplication                method    oa{sv}    -                                        -
.UnregisterApplication              method    o         -                                        -
org.bluez.LEAdvertisingManager1     interface -         -                                        -
.RegisterAdvertisement              method    oa{sv}    -                                        -
.UnregisterAdvertisement            method    o         -                                        -
.ActiveInstances                    property  y         0                                        emits-change
.SupportedIncludes                  property  as        3 "tx-power" "appearance" "local-name"   emits-change
.SupportedInstances                 property  y         5                                        emits-change
.SupportedSecondaryChannels         property  as        -                                        emits-change
org.bluez.Media1                    interface -         -                                        -
.RegisterApplication                method    oa{sv}    -                                        -
.RegisterEndpoint                   method    oa{sv}    -                                        -
.RegisterPlayer                     method    oa{sv}    -                                        -
.UnregisterApplication              method    o         -                                        -
.UnregisterEndpoint                 method    o         -                                        -
.UnregisterPlayer                   method    o         -                                        -
org.bluez.NetworkServer1            interface -         -                                        -
.Register                           method    ss        -                                        -
.Unregister                         method    s         -                                        -
org.freedesktop.DBus.Introspectable interface -         -                                        -
.Introspect                         method    -         s                                        -
org.freedesktop.DBus.Properties     interface -         -                                        -
.Get                                method    ss        v                                        -
.GetAll                             method    s         a{sv}                                    -
.Set                                method    ssv       -                                        -
.PropertiesChanged                  signal    sa{sv}as  -                                        -
```
There are a number that start with `org.bluez` which are documented at:
[https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc](https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc)

Those that start with `org.freedesktop.DBus` are standard interfaces and are
documented at: 
[https://dbus.freedesktop.org/doc/dbus-specification.html#standard-interfaces](https://dbus.freedesktop.org/doc/dbus-specification.html#standard-interfaces
)

## D-Bus bindings for Python <a name="bindings"></a>

There are a number of libraries that can be used to access D-Bus from Python.
However, they all seem to come with issues.

The BlueZ examples use `python-dbus` which the library accepts there might
be [issues](https://dbus.freedesktop.org/doc/dbus-python/). It is
documented that `python-dbus` is a legacy API, built with a deprecated 
dbus-glib library

The newer D-Bus libraries are based on functionality in `PyGObject` which uses
the D-Bus bindings in `gi.repsitory.Gio`.
However, this library is not documented well and I have only found a few
examples around.

`pydbus` has been a strong contender as it works great for doing 
BLE central devices, and it is built on `gi.repository.Gio`. However, 
it also has issues.
 - `pydbus` lacks support for `file descriptors`
 - clunky way of publishing required BlueZ information for servers 
 - Updates seem to have stopped happening on `pydbus`

Because of the issues stated above I've taken the decision to attempt to use
`PyGObject` module for more of my D-Bus work. Below are some notes of my
learnings from using it.

## Creating a proxy for a BlueZ object <a name="proxy"></a>

Object proxies are used as a way to create pythonic method calls to invoke
remote D-Bus methods.

As we have seen in the introspection above on the Bluetooth Adapter object,
there is a method called `GetDiscoveryFilters`. Taking the four pieces of 
information we know about bus type, bus name, object path, and interface we can
create a proxy and then call the method associated with that interface.

```python
from gi.repository import Gio

bus_type = Gio.BusType.SYSTEM
bus_name = 'org.bluez'
object_path = '/org/bluez/hci0'
interface = 'org.bluez.Adapter1'

adapter_proxy = Gio.DBusProxy.new_for_bus_sync(
            bus_type=bus_type,
            flags=Gio.DBusProxyFlags.NONE,
            info=None,
            name=bus_name,
            object_path=object_path,
            interface_name=interface,
            cancellable=None)

disco_filters = adapter_proxy.GetDiscoveryFilters()
print(disco_filters)
```
Which gives the output:
```python
['UUIDs', 'RSSI', 'Pathloss', 'Transport', 'DuplicateData', 'Discoverable']
```
## Discovering BlueZ Managed Objects <a name="interface"></a>

There will be a requirement for you to find the D-Bus object path for a piece
of Bluetooth information. For example, you may know the Bluetooth MAC address
of the device you want to interact with but not what its D-Bus object path is.

To find this information D-Bus has an `org.freedesktop.DBus.ObjectManager`
interface for recording the objects known about and BlueZ has implemented
this interface on the `/` object path. There is a method called 
`GetManagedObjects` which will return a dictionary of dictionaries.

For example to find the D-Bus object path for a device with an
address of `E1:4B:6C:22:56:F0` 
~~~ python
from gi.repository import Gio

bus_type = Gio.BusType.SYSTEM
bus_name = 'org.bluez'
object_path = '/'
mngr_iface = 'org.freedesktop.DBus.ObjectManager'
device_iface = 'org.bluez.Device1'
device_addr = 'E1:4B:6C:22:56:F0'

mngr_proxy = Gio.DBusProxy.new_for_bus_sync(
            bus_type=bus_type,
            flags=Gio.DBusProxyFlags.NONE,
            info=None,
            name=bus_name,
            object_path=object_path,
            interface_name=mngr_iface,
            cancellable=None)

mngd_objs = mngr_proxy.GetManagedObjects()
for obj_path, obj_data in mngd_objs.items():
    address = obj_data.get(device_iface, {}).get('Address')
    if address and address == device_addr:
        print(f'Device [{device_addr}] on object path: {obj_path}')

~~~
For me that gave the output of:
```text
Device [E1:4B:6C:22:56:F0] on object path: /org/bluez/hci0/dev_E1_4B_6C_22_56_F0
```

## Getting Properties <a name="properties"></a>

To interact with properties on an BlueZ interface we have to create a proxy
for the D-Bus standard interface of `org.freedesktop.DBus.Properties`
and use the `GetAll`, `Get` or `Set` methods which take the BlueZ
interface the property is on as input. The introspection above gives the
following signatures for the methods:
```text
NAME                                TYPE      SIGNATURE
.GetAll                             method    s
.Get                                method    ss
.Set                                method    ssv
```
The D-Bus documentation gives more information:
```text
    org.freedesktop.DBus.Properties.GetAll (in STRING interface_name,
                                          out ARRAY of DICT_ENTRY<STRING,VARIANT> props);
    org.freedesktop.DBus.Properties.Get (in STRING interface_name,
                                       in STRING property_name,
                                       out VARIANT value);
    org.freedesktop.DBus.Properties.Set (in STRING interface_name,
                                       in STRING property_name,
                                       in VARIANT value);
```
This is the first example where we have had inputs to our proxy methods calls.
`PyOBject` has the requirement that there is an extra input parameter which 
is the signature of the method.

Most of the inputs are stings which is straightforward taking a Python `str`
but in the case of the `Set` method we see that one of the parameters is of
D-Bus type `v`. 
This is `Variant` and we can create this using the `GLib` library.

An example to find all the properties on the Adapter interface, then
turn the adapter off, and back on again:

```python
from gi.repository import Gio, GLib

bus_type = Gio.BusType.SYSTEM
bus_name = 'org.bluez'
object_path = '/org/bluez/hci0'
prop_iface = 'org.freedesktop.DBus.Properties'
adapter_iface = 'org.bluez.Adapter1'


adapter_props_proxy = Gio.DBusProxy.new_for_bus_sync(
            bus_type=bus_type,
            flags=Gio.DBusProxyFlags.NONE,
            info=None,
            name=bus_name,
            object_path=object_path,
            interface_name=prop_iface,
            cancellable=None)

all_props = adapter_props_proxy.GetAll('(s)', adapter_iface)
print(all_props)
powered = adapter_props_proxy.Get('(ss)', adapter_iface, 'Powered')
print(powered)
adapter_props_proxy.Set('(ssv)',
                        adapter_iface,
                        'Powered',
                        GLib.Variant.new_boolean(False)
                        )
powered = adapter_props_proxy.Get('(ss)', adapter_iface, 'Powered')
print(powered)
adapter_props_proxy.Set('(ssv)',
                        adapter_iface,
                        'Powered',
                        GLib.Variant.new_boolean(True)
                        )
powered = adapter_props_proxy.Get('(ss)', adapter_iface, 'Powered')
print(powered)
```

This gave the output of:
```text
{'Address': 'FA:FB:FE:8F:1C:24', 'AddressType': 'public', 'Name': ...}
True
False
True
```

## Asynchronous Event Loop <a name="async"></a>

There is Bluetooth functionality that can be accessed synchronously such as
setting up properties on the adapter, connecting to a BLE device and even
reading a value from that device. There is other functionality that is designed
to work asynchronously such as events when a new device is found during 
scanning, a device connects, or a value on a device changes.

The PyOBject library uses an event loop called MainLoop. 

This will be the main loop in our program that typically waits for events
to trigger a callback function.

For example, let's implement the functionality to print out the time every
five seconds until we do a keyboard interrupt (ctrl-c). Typically, we might
do that with a while loop in Python:

```python
from datetime import datetime
from time import sleep


def show_time():
    now = datetime.now().strftime('%M:%S')
    print(f'Now: {now}')


try:
    while True:
        show_time()
        sleep(5)
except KeyboardInterrupt:
    print('exit')
```
To do this asynchronously we replace the while loop with the event loop.
Then set an event to occur at regular (5 seconds in this example) intervals.
We use the [timeout_add_seconds](https://lazka.github.io/pgi-docs/#GLib-2.0/functions.html#GLib.timeout_add_seconds)
method to add an event every 5 senconds. This event is to repeatedly call 
`show_time`. 
While `show_time` returns `True` the event will continue to automatically 
happen:

```python
from datetime import datetime
from gi.repository import GLib


def show_time():
    now = datetime.now().strftime('%M:%S')
    print(f'Now: {now}')
    return True


GLib.timeout_add_seconds(interval=5,
                         function=show_time)

mainloop = GLib.MainLoop()

try:
    mainloop.run()
except KeyboardInterrupt:
    print('exit')
    mainloop.quit()
```

## D-Bus Properties Changed <a name="properties_changed"></a>

As we can see above from the introspection of the adapter object path for the
`org.bluez.Adapter1` interface, the flag `emits-change` is set for all the 
properties. This means that if one or more of the properties change on the
object, the`org.freedesktop.DBus.Properties.PropertiesChanged` signal will
be emitted.
The PyOBject library does some work to make this easier by presenting the
[g-properties-changed](https://lazka.github.io/pgi-docs/Gio-2.0/classes/DBusProxy.html#Gio.DBusProxy.signals.g_properties_changed)
signal on our proxy. 
This signal corresponds to the`PropertiesChanged` D-Bus signal on the 
`org.freedesktop.DBus.Properties` interface.

This signal will happen when a property changes on interface. To use this
signal we use the proxy's `connect` method to link it with a function.

Below is an example of how this can be used. The example monitors the adapter
object and prints out when the status of the `Powered` property changes.

```python
from gi.repository import Gio, GLib

bus_type = Gio.BusType.SYSTEM
bus_name = 'org.bluez'
object_path = '/org/bluez/hci0'
adapter_iface = 'org.bluez.Adapter1'


def adapter_props_handler(proxy: Gio.DBusProxy,
                          changed_props: GLib.Variant,
                          invalidated_props: list) -> None:
    props = changed_props.unpack()
    powered = props.get('Powered')
    if powered is True:
        print('Adapter is now powered on')
    elif powered is False:
        print('Adapter is now powered off')


adapter_proxy = Gio.DBusProxy.new_for_bus_sync(
            bus_type=bus_type,
            flags=Gio.DBusProxyFlags.NONE,
            info=None,
            name=bus_name,
            object_path=object_path,
            interface_name=adapter_iface,
            cancellable=None)

adapter_proxy.connect('g-properties-changed', adapter_props_handler)
mainloop = GLib.MainLoop()

try:
    mainloop.run()
except KeyboardInterrupt:
    mainloop.quit()
```
This can be tested by, while the above Python script is running, using the
following on the command line:
```shell
$ bluetoothctl power off
$ bluetoothctl power on
```
The Python script should print:
```text
Adapter is now powered off
Adapter is now powered on
```

## Building a BLE Central <a name="ble_central"></a>

The above is enough D-Bus to create Bluetooth Low Energy (BLE) device to take
the Central role.
It hasn't covered scanning for devices and pairing (if required) but as those
are one-off provisioning steps they can be done with `bluetoothctl` on the 
command line.

The example below connects with a BBC micro:bit running its 
[UART service](https://lancaster-university.github.io/microbit-docs/resources/bluetooth/bluetooth_profile.html)
that has been configured to echo back any data that is sent to it.

```python
from time import sleep
from datetime import datetime
from gi.repository import Gio, GLib

DEVICE_ADDR = 'E1:4B:6C:22:56:F0'  # micro:bit address
UART_TX = '6e400002-b5a3-f393-e0a9-e50e24dcca9e'
UART_RX = '6e400003-b5a3-f393-e0a9-e50e24dcca9e'

# DBus Information
bus_type = Gio.BusType.SYSTEM
BLUEZ_NAME = 'org.bluez'
ADAPTER_PATH = '/org/bluez/hci0'
device_path = f"{ADAPTER_PATH}/dev_{DEVICE_ADDR.replace(':', '_')}"
MNGR_IFACE = 'org.freedesktop.DBus.ObjectManager'
PROP_IFACE = 'org.freedesktop.DBus.Properties'
DEVICE_IFACE = 'org.bluez.Device1'
BLE_CHRC_IFACE = 'org.bluez.GattCharacteristic1'


def bluez_proxy(object_path, interface):
    return Gio.DBusProxy.new_for_bus_sync(
        bus_type=bus_type,
        flags=Gio.DBusProxyFlags.NONE,
        info=None,
        name=BLUEZ_NAME,
        object_path=object_path,
        interface_name=interface,
        cancellable=None)


# setup dbus
mngr = bluez_proxy('/', MNGR_IFACE)
device = bluez_proxy(device_path, DEVICE_IFACE)
dev_props = bluez_proxy(device_path, PROP_IFACE)

# Connect to device
device.Connect()

# Wait for the cache of services on the remote device to be updated
while not dev_props.Get('(ss)', DEVICE_IFACE, 'ServicesResolved'):
    print(f"Services Resolved: {dev_props.Get('(ss)', DEVICE_IFACE, 'ServicesResolved')}")
    sleep(0.5)
print(f"Services Resolved: {dev_props.Get('(ss)', DEVICE_IFACE, 'ServicesResolved')}")


def get_characteristic_path(dev_path, uuid):
    """Look up DBus path for characteristic UUID"""
    mng_objs = mngr.GetManagedObjects()
    for path in mng_objs:
        chr_uuid = mng_objs[path].get(BLE_CHRC_IFACE, {}).get('UUID')
        if path.startswith(dev_path) and chr_uuid == uuid.casefold():
            return path


# Characteristic DBus information
tx_path = get_characteristic_path(device_path, UART_TX)
tx_proxy = bluez_proxy(tx_path, BLE_CHRC_IFACE)
rx_path = get_characteristic_path(device_path, UART_RX)
rx_proxy = bluez_proxy(rx_path, BLE_CHRC_IFACE)

rx_proxy.WriteValue('(aya{sv})', b'Test#', {})


def tx_handler(proxy: Gio.DBusProxy,
                changed_props: GLib.Variant,
                invalidated_props: list) -> None:
    """Notify event handler for messages received from UART Service"""
    props = changed_props.unpack()
    value = props.get('Value')
    if value:
        text = bytes(value).strip()
        print(f'Echo: {text}')


def send_time():
    """Send time over UART. `#` is the message termination character"""
    now = datetime.now().strftime('%M:%S#')
    print(f'Send: {now.encode()}')
    rx_proxy.WriteValue('(aya{sv})', now.encode(), {})
    return True


# Enable eventloop for notifications
mainloop = GLib.MainLoop()
GLib.timeout_add_seconds(interval=9, function=send_time)
tx_proxy.connect('g-properties-changed', tx_handler)
tx_proxy.StartNotify()

try:
    mainloop.run()
except KeyboardInterrupt:
    mainloop.quit()
    tx_proxy.StopNotify()
    device.Disconnect()
```

---

&copy; Copyright 2022, Barry Byford.

first published: 2022 January 21

last updated: 2022 January 23

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
