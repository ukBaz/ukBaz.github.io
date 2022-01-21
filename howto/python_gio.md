# D-Bus and Bluez

## Overview
If you want to do Bluetooth on Linux then BlueZ (the official Bluetooth stack
on linux) is the best option. They have made a number of APIs available for
people to interface with the functionality and write their applications. These
APIs use D-Bus which is not widely known about so this article aims to give
some background to help people get started with writing applications using
BlueZ.

## D-Bus
D-Bus allows communication between multiple processes running concurrently 
on the same machine. In the case of BlueZ this is between the Bluetooth
Daemon (`bluetoothd`) and the application you have written.

An advantage of D-Bus is that most programming languages have bindings to it
so the BlueZ APIs are language agnostic. D-Bus bindings require you to know
four bits of information:
1) Which bus (`system` or `session`) the service is on
2) The `bus name` of the service
3) The `object path` of the methods and properties
4) The `interface` on that `object path`
5) `method`, `property` or `signal` 

This hierarchy of information builds to uniquely identify what is to be
accessed with D-Bus.

    System Bus -> Bus Name -> Object Path -> Interface -> Method

Let's look at each of these in more detail.

### Buses
There are two buses:
 - a `session` bus for each user login session
 - a single `system` bus that provides access to system services

For BlueZ and accessing `bluetoothd` we will always use the `system` bus

### Bus Name
To see all the `system` D-Bus names available on your machine run the following
command line:
```shell
$ busctl list
```
In the list will be `org.bluez` and this is what is used for BlueZ information.

### Object Path
The `object path` looks like a filesystem path, but they are not, they are 
identifiers of unique objects on the D-Bus.
For example the default Bluetooth adapter on a Linux system is typically
`/org/bluez/hci0`
A list of all the objects being managed by BlueZ can be got on the command line
with the following command:
```shell
$ busctl tree org.bluez
└─/org
  └─/org/bluez
    └─/org/bluez/hci0
      └─/org/bluez/hci0/dev_12_34_56_78_9A_BC
        └─/org/bluez/hci0/dev_12_34_56_78_9A_BC/service000a
          └─/org/bluez/hci0/dev_12_34_56_78_9A_BC/service000a/char000b
            └─/org/bluez/hci0/dev_12_34_56_78_9A_BC/service000a/char000b/desc000d

```

### Interface
There may be a number of interfaces on an object path. For example on 
`/org/bluez/hci0`

    org.bluez.Adapter1
    org.bluez.GattManager1
    org.bluez.LEAdvertisingManager1
    org.bluez.Media1
    org.bluez.NetworkServer1
    org.freedesktop.DBus.Introspectable
    org.freedesktop.DBus.Properties

We can find out what is on the object by introspection. Here is an example:
```shell
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
https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc

Those that start with `org.freedesktop.DBus` are standard interfaces and are
documented at: 
https://dbus.freedesktop.org/doc/dbus-specification.html#standard-interfaces

## D-Bus bindings for Python
There are a number of libraries that can be used to access D-Bus from Python.
However they all seem to come with issues.

The BlueZ examples use `python-dbus` which the library accepts there might
be [issues](https://dbus.freedesktop.org/doc/dbus-python/). Else where it is
documented that `dbus-python` is a legacy API, built with a deprecated 
dbus-glib library

The newer D-Bus libraries are based on functionality in `PyGObject` which uses
the D-Bus bindings in `gi.repsitory.Gio`.
However, this library is not documented well and only minimal examples
around.

`pydbus` has been a strong contender as it works great for doing 
BLE central devices, and it is built on `gi.repository.Gio`. However, 
it also has issues.
 - `pydbus` lacks support for `file descriptors`
 - clunky way of publishing required BlueZ information for servers 
 - Updates seem to have stopped happening on `pydbus`

Because of the issues stated above I've taken the decision to attempt to use
`PyGObject` module for more of D-Bus work. Below is some of those learnings.

## Creating a proxy for a BlueZ object
Object proxies are used as a way to pythonic method calls to invoke remote
D-Bus methods.

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
## Getting BlueZ Managed Objects
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

# Getting Properties
To interact with properties on an BlueZ interface we have to create a proxy
for the D-Bus standard interface of `org.freedesktop.DBus.Properties`
and use the `GetAll`, `Get` or `Set` methods giving them the BlueZ
interface the property is on. The introspection above gives what the
signature is of the methods
```text
NAME                                TYPE      SIGNATURE
.GetAll                             method    s
.Get                                method    ss
.Set                                method    ssv
```
The D-Bus tells
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
This is the first example where we have had inputs to our methods. `PyOBject`
has the requirement that there is an extra input parameter. This extra 
parameter goes first and is the signature of the method.

In the case of the `Set` method we see that one of the parameters is of D-Bus
type `v`. This is `Variant` and we can set this using the `Glib` library.

An example to find all the properties on the Adapter interface and then
turn the adapter off and back on again:

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

---

&copy; Copyright 2022, Barry Byford.

last updated: 2022 January 21

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
