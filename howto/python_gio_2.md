# Beacon Scanner

## Introduction

In [part 1](python_gio_1.html) of this series of posts we looked at the basics
of working with D-Bus and BlueZ. In this part we take on the tricky topic
of scanning for nearby devices. A typical example of an application for this
is to scan for nearby BLE Beacons.

Bluetooth Low Energy (BLE) Beacon is a device that only transmits small 
amounts of data and usually can not be connected to. iBeacon and Eddystone
are two of the more well known formats. The small amount of data is stored
in either the `service data` or `manufacturer data` property depending
on the beacon.

## BlueZ D-Bus Object Creation Signal

The core of this functionality from PyGObject is the
[DBusObjectManagerClient](https://lazka.github.io/pgi-docs/index.html#Gio-2.0/classes/DBusObjectManager.html)
functionality.

Once an instance of this type has been created, you can connect to the 
`Gio.DBusObjectManager` [object-added](https://lazka.github.io/pgi-docs/index.html#Gio-2.0/classes/DBusObjectManager.html#Gio.DBusObjectManager.signals.object_added)
signal.
This enables having callbacks that are run everytime a new BlueZ object is
added.

In the example below a class is created to inherit from `GObject.GObject` so
that custom signals can be created for when specific BlueZ objects are created.
In this case it is an adapter and a device being added. This isn't the only
way of doing this. It could be that the `_on_object_added` could call the
`new_device_hndlr` function directly. In this example it slightly complicates
things but if it was part of a bigger library and there was a need for
multiple listners then it could be more useful.

## Filtering of objects

As the device is never connected to there is no need to store the device in
the BlueZ cache so the example removes the devices once it has read the 
property information in the advertisement.

This is a good thing to do for beacons for a couple of reasons. First is that
the one the device is in the cache it is the 
[g-properties-changed](python_gio_1.html#properties_changed) 
signal on the device proxy that needs to be monitored.
Secondly, once the device is in the cache the Linux kernel does some filtering
of that device to ensure it doesn't spam the system with notifications.
Typically, for beacon scanning applications it is desirable to see every
advertisement from the beacon.

## Signals and Variants

To make the `device-added` signal includes all the device properties as a
dictionary. In D-Bus the values in the dictionary need to be converted into
`variant` types. This is done with the `create_variant` function. Rather than
guess the D-Bus type from the Python type, this function looks up the property
type. 

```python
import logging
from gi.repository import Gio, GLib, GObject

# DBus Information
bus_type = Gio.BusType.SYSTEM
BLUEZ_NAME = 'org.bluez'
ADAPTER_PATH = '/org/bluez/hci0'
PROP_IFACE = 'org.freedesktop.DBus.Properties'
ADAPTER_IFACE = 'org.bluez.Adapter1'
DEVICE_IFACE = 'org.bluez.Device1'

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger('scan_app')


def create_variant(py_data):
    """
    convert python native data types to D-Bus variant types by looking up
    their type expected for that key.
    """
    type_lookup = {'Address': 's',
                   'AddressType': 's',
                   'Name': 's',
                   'Icon': 's',
                   'Class': 'u',
                   'Appearance': 'q',
                   'Alias': 's',
                   'Paired': 'b',
                   'Trusted': 'b',
                   'Blocked': 'b',
                   'LegacyPairing': 'b',
                   'RSSI': 'n',
                   'Connected': 'b',
                   'UUIDs': 'as',
                   'Adapter': 'o',
                   'ManufacturerData': 'a{qay}',
                   'ServiceData': 'a{say}',
                   'TxPower': 'n',
                   'ServicesResolved': 'b',
                   'WakeAllowed': 'b',
                   'Modalias': 's',
                   'AdvertisingFlags': 'ay',
                   'AdvertisingData': 'a{yay}',
                   'Powered': 'b',
                   'Discoverable': 'b',
                   'Pairable': 'b',
                   'PairableTimeout': 'u',
                   'DiscoverableTimeout': 'u',
                   'Discovering': 'b',
                   'Roles': 'as',
                   'ExperimentalFeatures': 'as',
                   }
    if py_data is None:
        return GLib.Variant('a{sv}', {})
    for k, v in py_data.items():
        py_data[k] = GLib.Variant(type_lookup[k], v)
    return GLib.Variant('a{sv}', py_data)


class BluezObjectManager(GObject.GObject):
    __gsignals__ = {
        'adapter-added': (GObject.SignalFlags.NO_HOOKS, None,
                          (str, GObject.TYPE_VARIANT)),
        'adapter-removed': (GObject.SignalFlags.NO_HOOKS, None, (str,)),
        'device-added': (GObject.SignalFlags.NO_HOOKS, None,
                         (str, GObject.TYPE_VARIANT)),
        'device-removed': (GObject.SignalFlags.NO_HOOKS, None, (str,)),
    }

    def __init__(self) -> None:
        super().__init__()
        self._object_manager = Gio.DBusObjectManagerClient.new_for_bus_sync(
            bus_type,
            Gio.DBusObjectManagerClientFlags.DO_NOT_AUTO_START,
            BLUEZ_NAME,
            '/', None, None, None)

        self._object_manager.connect("object-added", self._on_object_added)
        self._object_manager.connect("object-removed", self._on_object_removed)

    def _on_object_added(self,
                         _object_manager: Gio.DBusObjectManager,
                         dbus_object: Gio.DBusObject) -> None:
        object_path = dbus_object.get_object_path()
        ifaces = [iface.get_interface_name()
                  for iface in dbus_object.get_interfaces()]
        if ADAPTER_IFACE in ifaces:
            prop_proxy = dbus_object.get_interface(PROP_IFACE)
            adapter_props = prop_proxy.GetAll('(s)', ADAPTER_IFACE)
            logger.debug('adapter-added %s', object_path, adapter_props)
            self.emit('adapter-added', object_path, adapter_props)
        elif DEVICE_IFACE in ifaces:
            prop_proxy = dbus_object.get_interface(PROP_IFACE)
            dev_props = prop_proxy.GetAll('(s)', DEVICE_IFACE)
            logger.debug('device-added %s : %s', object_path, dev_props)
            self.emit('device-added', object_path, create_variant(dev_props))

    def _on_object_removed(self,
                           _object_manager: Gio.DBusObjectManager,
                           dbus_object: Gio.DBusObject) -> None:
        object_path = dbus_object.get_object_path()
        ifaces = [iface.get_interface_name()
                  for iface in dbus_object.get_interfaces()]
        if ADAPTER_IFACE in ifaces:
            logger.debug('adapter-removed %s : %s', object_path)
            self.emit('adapter-removed', object_path)
        elif DEVICE_IFACE in ifaces:
            logger.debug('device-removed: %s', object_path)
            self.emit('device-removed', object_path)


def bluez_proxy(object_path, interface):
    """Return a BlueZ proxy object for the given D-Bus information"""
    return Gio.DBusProxy.new_for_bus_sync(
        bus_type=bus_type,
        flags=Gio.DBusProxyFlags.NONE,
        info=None,
        name=BLUEZ_NAME,
        object_path=object_path,
        interface_name=interface,
        cancellable=None)


def new_device_hndlr(proxy: BluezObjectManager,
                     object_path: str,
                     device_props: GObject.TYPE_VARIANT) -> None:
    """Event handler for New device has been detected with scan"""
    props = device_props.unpack()
    logger.debug('New Device Handler: %s : %s', object_path, props)
    address = props.get('Address')
    if address:
        print(f'Device with address {address} found. '
              f'Removing from BlueZ cache')
    # adapter.RemoveDevice('(o)', GLib.Variant.new_object_path(object_path))
    adapter.RemoveDevice('(o)', object_path)


def stop_scan():
    """Stop scanning for new devices and quit event loop"""
    logger.info('Stopping Discovery')
    adapter.StopDiscovery()
    mainloop.quit()
    return False


if __name__ == '__main__':
    # setup dbus
    mngr = BluezObjectManager()
    adapter = bluez_proxy(ADAPTER_PATH, ADAPTER_IFACE)
    # Link device-added event to callback function
    mngr.connect('device-added', new_device_hndlr)
    # Start discovering (scanning) for devices
    adapter.StartDiscovery()

    # Enable eventloop for notifications
    mainloop = GLib.MainLoop()
    # Create a timed event to call the stop_scan function
    GLib.timeout_add_seconds(interval=20, function=stop_scan)

    try:
        mainloop.run()
    except KeyboardInterrupt:
        mainloop.quit()
        adapter.StopDiscovery()
```
---

&copy; Copyright 2022, Barry Byford.

first published: 2022 January 29

last updated: 2022 January 29

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
