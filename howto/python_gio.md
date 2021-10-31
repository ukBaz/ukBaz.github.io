Using gi.repository for D-Bus
=============================

This article is under development as I look for alternatives python bindings
to D-Bus to use the API from BlueZ.

The BlueZ examples use `python-dbus` but that library has been deprecated.

`pydbus` has been a strong contender. It works great for doing BLE central
devices. However it has issues.
 - its lack of support for `file descriptors`
 - clunky way of publishing required BlueZ information for servers 
 - Updates seem to have stopped happening on `pydbus`

The new D-Bus libraries are based on functionality in `gi.repsitory.Gio`.
It is not documented well and there are not many examples around. So I'm
wading my way through it to see if I can pull out a few key examples to make
it a good choice going forward.

# Getting BlueZ Managed Objects
~~~ python
from gi.repository import Gio, GLib

def get_managed_objects():
    return Gio.DBusProxy.new_for_bus_sync(
            bus_type=Gio.BusType.SYSTEM,
            flags=Gio.DBusProxyFlags.NONE,
            info=None,
            name='org.bluez',
            object_path='/',
            interface_name='org.freedesktop.DBus.ObjectManager',
            cancellable=None).GetManagedObjects()
~~~

# Calling Methods and Getting Properties

~~~ python
from gi.repository import Gio

con = Gio.DBusObjectManagerClient.new_for_bus_sync(
            bus_type=Gio.BusType.SYSTEM,
            flags=Gio.DBusObjectManagerClientFlags.NONE,
            name='org.bluez',
            object_path='/',
            get_proxy_type_func=None,
            get_proxy_type_user_data=None,
            cancellable=None,
        )

adapter_methods = con.get_interface('/org/bluez/hci0',
                                    'org.bluez.Adapter1')
adapter_properties = con.get_interface('/org/bluez/hci0',
                                       'org.freedesktop.DBus.Properties')

powered = adapter_properties.Get('(ss)', 'org.bluez.Adapter1', 'Powered')
print(f'Powered = {powered}')
# Powered = True
~~~