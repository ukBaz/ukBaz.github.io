# Publish To D-Bus

This is part of a series of posts about using PyGObject for D-Bus with
a specific focus on BlueZ. Parts 
[one](python_gio_1.html) 
and 
[two](python_gio_2.html) 
on this topic could be worth reading before this post if you are 
new to the topic.

## Introduction


BlueZ uses a technique where an interface is defined 
(e.g. `org.bluez.LEAdvertisement1`) for the developer to create that interface
and publish it to D-Bus. The location of that create interface implementation 
is given to BlueZ using the relevant `Register*` command 
(e.g. `RegisterAdvertisement` on the `org.bluez.LEAdvertisingManager1` 
interface).

Examples of the various `Register*` commands:

```text
advertising-api.txt:RegisterAdvertisement(object advertisement, dict options)
agent-api.txt:RegisterAgent(object agent, string capability)
gatt-api.txt:RegisterApplication(object application, dict options)
media-api.txt:RegisterEndpoint(object endpoint, dict properties)
media-api.txt:RegisterPlayer(object player, dict properties)
media-api.txt:RegisterApplication(object root, dict options)
obex-agent-api.txt:RegisterAgent(object agent)
profile-api.txt:RegisterProfile(object profile, string uuid, dict options)
```

This is why the interfaces that need to be implemented are documented as:

```text
Object path:	freely definable
```

Because the relevant `Register*` command will be used to tell BlueZ the
object path of the interface that has been implemented.

To state this a different way, BlueZ provides a blueprint for D-Bus services
that it needs to do certain things. e.g. advertising or an agent.
The documented interface has no implementation. 
This is created for the specific instance (e.g. what it is we want to 
advertise). BlueZ is told where to find this interface implementation with the
specific `Register*` command.


## Is Writing D-Bus Service with PyGObject broken?

A reading of the documentation suggests the way to create a 
D-Bus service with PyGObject is with 
[DBusInterfaceSkeleton](https://lazka.github.io/pgi-docs/Gio-2.0/classes/DBusInterfaceSkeleton.html)
but after much experimentation I came to the same conclusion as outlined in the
following StackOverflow [post](https://stackoverflow.com/a/54599507/7721752),
that it does not work for the Python bindings.

There does appear to be a way to do this with 
[closures](https://en.wikipedia.org/wiki/Closure_(computer_programming))
rather than
[virtual table methods](https://en.wikipedia.org/wiki/Virtual_method_table)

## Publishing a D-Bus Service

Let's start with a non-BlueZ example in the hope that it will be shorter
and simpler to understand.

In the example below `DbusService` is the class for handling the transfer
between python methods & properties and published D-Bus methods & properties.
For the purposes of keeping this blog shorter than it might otherwise be 
I'm not going to explain the details here. The class will probably will 
not work in all situations but for working with BlueZ it is good enough.

`MyDBusService` is an example of a specific implementation of D-Bus service 
that is to be published. 
The `introspection_xml` contains the D-Bus introspection information.
The node information is built from this xml data using 
[DBusNodeInfo.new_for_xml](https://lazka.github.io/pgi-docs/index.html#Gio-2.0/classes/DBusNodeInfo.html#Gio.DBusNodeInfo.new_for_xml)
The methods and properties specified match a Python method and 
property created in this class.
The  XML 
[introspection format](https://dbus.freedesktop.org/doc/dbus-specification.html#introspection-format)
is described in the D-Bus specification

`DbusService` is doing the majority of the heavy lifting while `MyDBusService`
focuses on the service to be published.

The created service needs to wait in an event loop for calls to it. Again,
it is `MainLoop` that is used to create this.

```python
import logging
from math import sqrt

from gi.repository import Gio, GLib, GObject

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger('calc')


def _build_variant(name, py_value):
    s_data = GLib.VariantDict.new()
    for key, value in py_value.items():
        gvalue = GLib.Variant('ay', value)
        s_data.insert_value(key, gvalue)
    return s_data.end()


class DbusService:

    def __init__(self, introspection_xml, publish_path, 
                 own_name=None, sys_bus=True):
        self.node_info = Gio.DBusNodeInfo.new_for_xml(introspection_xml).interfaces[0]
        method_outargs = {}
        method_inargs = {}
        property_sig = {}
        for method in self.node_info.methods:
            method_outargs[method.name] = '(' + ''.join([arg.signature for arg in method.out_args]) + ')'
            method_inargs[method.name] = tuple(arg.signature for arg in method.in_args)
        self.method_inargs = method_inargs
        self.method_outargs = method_outargs
        if sys_bus:
            self.con = Gio.bus_get_sync(Gio.BusType.SYSTEM, None)
        else:
            self.con = Gio.bus_get_sync(Gio.BusType.SESSION, None)
        if own_name:
            Gio.bus_own_name_on_connection(connection=self.con,
                                           name=own_name,
                                           flags=Gio.BusNameOwnerFlags.NONE,
                                           name_acquired_closure=None,
                                           name_lost_closure=None)
        self.con.register_object(
            publish_path,
            self.node_info,
            self.handle_method_call,
            self.prop_getter,
            self.prop_setter)

    def handle_method_call(self,
                           connection: Gio.DBusConnection,
                           sender: str,
                           object_path: str,
                           interface_name: str,
                           method_name: str,
                           params: GLib.Variant,
                           invocation: Gio.DBusMethodInvocation
                           ):
        """
        This is the top-level function that handles method calls to
        the server.
        """
        args = list(params.unpack())
        for i, sig in enumerate(self.method_inargs[method_name]):
            # Check if there is a Unix file descriptor  in the signature
            if sig == 'h':
                msg = invocation.get_message()
                fd_list = msg.get_unix_fd_list()
                args[i] = fd_list.get(args[i])
        # Get the method from the Python class
        func = self.__getattribute__(method_name)
        result = func(*args)
        if result is None:
            result = ()
        else:
            result = (result,)
        outargs = ''.join([_.signature
                           for _ in invocation.get_method_info().out_args])
        send_result = GLib.Variant(f'({outargs})', result)
        logger.debug('Method %s result: %s', method_name, repr(send_result))
        invocation.return_value(send_result)

    def prop_getter(self,
                    connection: Gio.DBusConnection,
                    sender: str,
                    object: str,
                    iface: str,
                    name: str):
        """Mehtod for moving properties from Python Class to D-Bus"""
        logger.debug('prop_getter, %s, %s, %s, %s, %s',
                     connection, sender, object, iface, name)
        py_value = self.__getattribute__(name)
        signature = self.node_info.lookup_property(name).signature
        if 'v' in signature:
            dbus_value = _build_variant(name, py_value)
            return dbus_value
        if py_value:
            return GLib.Variant(signature, py_value)
        return None

    def prop_setter(self,
                    connection: Gio.DBusConnection,
                    sender: str,
                    object: str,
                    iface: str,
                    name: str,
                    value: GLib.Variant):
        """Method for moving properties between D-Bus and Python Class"""
        logger.debug('prop_setter %s, %s, %s, %s, %s, %s',
                     connection, sender, object, iface, name, value)
        self.__setattr__(name, value.unpack())
        return True


class MyDBusService(DbusService):
    introspection_xml = """
        <!DOCTYPE node PUBLIC
        "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
        "http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
        <node>
            <interface name="org.ukbaz.calculator1">
                <method name="Fibonacci">
                    <arg type="q" name="position" direction="in"/>
                    <arg type="t" name="result" direction="out"/>
                </method>
                <property name="limit" type="t" access="read"/>
            </interface>
        </node>
        """

    def __init__(self):
        self.path = '/org/ukbaz/fibonacci0001'
        super().__init__(self.introspection_xml, self.path, 'org.ukbaz')

        self.limit = 94

    def Fibonacci(self, position):
        return round(
            (((1 + sqrt(5)) / 2) ** position - (
                    (1 - sqrt(5)) / 2) ** position) / sqrt(5))


def main():
    service = MyDBusService()
    mainloop = GLib.MainLoop()
    try:
        mainloop.run()
    except KeyboardInterrupt:
        mainloop.quit()


if __name__ == '__main__':
    main()
```

This can be tested from the Linux command line using the `busctl` command.

For example to call the `Fibonacci` method:
```text
$ busctl --user call org.ukbaz /org/ukbaz/fibonacci0001 org.ukbaz.calculator1 Fibonacci q 9
t 34
```

Or retrieve the `limit` property:

```text
$ busctl --user get-property org.ukbaz /org/ukbaz/fibonacci0001 org.ukbaz.calculator1 limit
t 94
```

This experiment has used the `session` (or `user`) bus to simplify the 
security setup to specify name of our service.

For BlueZ we have to use the `system` bus. Because we are creating the 
service and registering it with BlueZ we do not need to give it a nice
human-readable name.

## Building An Bluetooth Advertisement

To create an advertisement in BlueZ with the D-Bus API there are two things 
that need to happen. The first is that a D-Bus service with the interface
`org.bluez.LEAdvertisement1` and the second is to tell BlueZ where that service
is with `RegisterAdvertisement`.

### Defining the Interface

There needs to be an XML definition of the interface that is used for defining
the introspection details of the interface. This information can be stored in
a separate file, doc string, or a variable. It needs to be accessible to read.

```xml
<!DOCTYPE node PUBLIC
"-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
"http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
<node>
    <interface name="org.bluez.LEAdvertisement1">
        <method name="Release"/>
        <property name="Type" type="s" access="readwrite"/>
        <property name="ServiceUUIDs" type="as" access="readwrite"/>
        <property name="ManufacturerData" type="a{sv}" access="readwrite"/>
        <property name="SolicitUUIDs" type="as" access="readwrite"/>
        <property name="ServiceData" type="a{sv}" access="readwrite"/>
        <property name="Includes" type="as" access="readwrite"/>
        <property name="LocalName" type="s" access="readwrite"/>
        <property name="Appearance" type="q" access="readwrite"/>
        <property name="Duration" type="q" access="readwrite"/>
        <property name="Timeout" type="q" access="readwrite"/>
    </interface>
</node>
```
### BlueZ Advertisement example

Below is an example of creating an Eddystone URL beacon. This is a lot longer
than the previous example. That main increase in length and complexity
is from manipulating data between Python and D-Bus

```python
import logging
import threading
from typing import List, Dict, Union

from gi.repository import Gio, GLib, GObject

# DBus Information
bus_type = Gio.BusType.SYSTEM
BLUEZ_NAME = 'org.bluez'
ADAPTER_PATH = '/org/bluez/hci0'
PROP_IFACE = 'org.freedesktop.DBus.Properties'
ADAPTER_IFACE = 'org.bluez.Adapter1'
DEVICE_IFACE = 'org.bluez.Device1'
# BlueZ DBus Advertising Manager Interface
LE_ADVERTISING_MANAGER_IFACE = 'org.bluez.LEAdvertisingManager1'
# BlueZ DBus Advertisement Interface
LE_ADVERTISEMENT_IFACE = 'org.bluez.LEAdvertisement1'

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger('advert')

introspection_xml = """
    <!DOCTYPE node PUBLIC
    "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
    "http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
    <node>
        <interface name="org.bluez.LEAdvertisement1">
            <method name="Release"/>
            <property name="Type" type="s" access="readwrite"/>
            <property name="ServiceUUIDs" type="as" access="readwrite"/>
            <property name="ManufacturerData" type="a{sv}" access="readwrite"/>
            <property name="SolicitUUIDs" type="as" access="readwrite"/>
            <property name="ServiceData" type="a{sv}" access="readwrite"/>
            <property name="Includes" type="as" access="readwrite"/>
            <property name="LocalName" type="s" access="readwrite"/>
            <property name="Appearance" type="q" access="readwrite"/>
            <property name="Duration" type="q" access="readwrite"/>
            <property name="Timeout" type="q" access="readwrite"/>
        </interface>
    </node>
    """


def _build_variant(name, py_data):
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
    logger.debug('Create variant(%s, %s)', name, py_data)
    return GLib.Variant(type_lookup[name], py_data)


def _build_variant2(name, py_value):
        s_data = GLib.VariantDict.new()
        for key, value in py_value.items():
            gvalue = GLib.Variant('ay', value)
            s_data.insert_value(key, gvalue)
        return s_data.end()


def bluez_proxy(object_path, interface):
    """Create a BlueZ proxy object"""
    return Gio.DBusProxy.new_for_bus_sync(
        bus_type=bus_type,
        flags=Gio.DBusProxyFlags.NONE,
        info=None,
        name=BLUEZ_NAME,
        object_path=object_path,
        interface_name=interface,
        cancellable=None)


class DbusService:

    def __init__(self, introspection_xml, publish_path,
                 own_name=None, sys_bus=True):
        self.node_info = Gio.DBusNodeInfo.new_for_xml(introspection_xml).interfaces[0]
        method_outargs = {}
        method_inargs = {}
        property_sig = {}
        for method in self.node_info.methods:
            method_outargs[method.name] = '(' + ''.join(
                [arg.signature
                 for arg in method.out_args]) + ')'
            method_inargs[method.name] = tuple(arg.signature
                                               for arg in method.in_args)
        self.method_inargs = method_inargs
        self.method_outargs = method_outargs
        if sys_bus:
            self.con = Gio.bus_get_sync(Gio.BusType.SYSTEM, None)
        else:
            self.con = Gio.bus_get_sync(Gio.BusType.SESSION, None)
        if own_name:
            Gio.bus_own_name_on_connection(connection=self.con,
                                           name=own_name,
                                           flags=Gio.BusNameOwnerFlags.NONE,
                                           name_acquired_closure=None,
                                           name_lost_closure=None)
        self.con.register_object(
            publish_path,
            self.node_info,
            self.handle_method_call,
            self.prop_getter,
            self.prop_setter)

    def handle_method_call(self,
                           connection: Gio.DBusConnection,
                           sender: str,
                           object_path: str,
                           interface_name: str,
                           method_name: str,
                           params: GLib.Variant,
                           invocation: Gio.DBusMethodInvocation
                           ):
        """
        This is the top-level function that handles method calls to
        the server.
        """
        args = list(params.unpack())
        for i, sig in enumerate(self.method_inargs[method_name]):
            # Check if there is a Unix file descriptor  in the signature
            if sig == 'h':
                msg = invocation.get_message()
                fd_list = msg.get_unix_fd_list()
                args[i] = fd_list.get(args[i])
        # Get the method from the Python class
        func = self.__getattribute__(method_name)
        result = func(*args)
        if result is None:
            result = ()
        else:
            result = (result,)
        outargs = ''.join([_.signature
                           for _ in invocation.get_method_info().out_args])
        send_result = GLib.Variant(f'({outargs})', result)
        logger.debug('Method Call result: %s', repr(send_result))
        invocation.return_value(send_result)

    def prop_getter(self,
                    connection: Gio.DBusConnection,
                    sender: str,
                    object: str,
                    iface: str,
                    name: str):
        """Mehtod for moving properties from Python Class to D-Bus"""
        logger.debug('prop_getter, %s, %s, %s, %s, %s',
                     connection, sender, object, iface, name)
        py_value = self.__getattribute__(name)
        signature = self.node_info.lookup_property(name).signature
        if 'v' in signature:
            print('py_value', py_value)
            dbus_value = _build_variant2(name, py_value)
            print('dbus_value', dbus_value)
            return dbus_value
        if py_value:
            return GLib.Variant(signature, py_value)
        return None

    def prop_setter(self,
                    connection: Gio.DBusConnection,
                    sender: str,
                    object: str,
                    iface: str,
                    name: str,
                    value: GLib.Variant):
        """Method for moving properties between D-Bus and Python Class"""
        logger.debug('prop_setter %s, %s, %s, %s, %s, %s',
                     connection, sender, object, iface, name, value)
        # x_value = GLib.Variant('as', ['test'])
        self.__setattr__(name, value.unpack())
        return True


class Advertisement(DbusService):
    """Advertisement data"""

    def __init__(self, advert_id, ad_type):
        # Setup D-Bus object paths
        self.path = '/org/bluez/advertisement{0:04d}'.format(advert_id)
        super().__init__(introspection_xml=introspection_xml,
                         publish_path=self.path)

        self.Type = ad_type
        self.ServiceUUIDs = []
        self.ManufacturerData = {}
        self.SolicitUUIDs = []
        self.ServiceData = {}
        self.Includes = []
        self.LocalName = None
        self.Appearance = None
        self.Duration = None
        self.Timeout = None

        self.mainloop = GLib.MainLoop()
        self._ad_thread = None

    def _publish(self):
        self.mainloop.run()

    def start(self):
        """Start GLib event loop"""
        self._ad_thread = threading.Thread(target=self._publish)
        self._ad_thread.daemon = True
        self._ad_thread.start()

    def stop(self):
        """Stop GLib event loop"""
        self.mainloop.quit()

    def Release(self):  # pylint: disable=invalid-name
        """
        This method gets called when the service daemon
        removes the Advertisement. A client can use it to do
        cleanup tasks. There is no need to call
        UnregisterAdvertisement because when this method gets
        called it has already been unregistered.
        :return:
        """
        pass

    @property
    def service_UUIDs(self):  # pylint: disable=invalid-name
        """List of UUIDs that represent available services."""
        return self.ServiceUUIDs.unpack()

    @service_UUIDs.setter
    def service_UUIDs(self, UUID):  # pylint: disable=invalid-name
        self.ServiceUUIDs = GLib.Variant('as', UUID)

    @property
    def manufacturer_data(self, company_id, data):
        """Manufacturer Data to be broadcast"""
        return self.ManufacturerData.unpack()

    @manufacturer_data.setter
    def manufacturer_data(self, manufacturer_data: Dict[int, List[int]]) -> None:
        """Manufacturer Data to be broadcast"""
        m_data = GLib.VariantBuilder(GLib.VariantType.new('a{qv}'))
        for key, value in manufacturer_data.items():
            g_key = GLib.Variant.new_uint16(key)
            g_value = GLib.Variant('ay', value)
            g_var = GLib.Variant.new_variant(g_value)
            g_dict = GLib.Variant.new_dict_entry(g_key, g_var)
            m_data.add_value(g_dict)
        self.ManufacturerData = m_data.end()

    @property
    def solicit_UUIDs(self):  # pylint: disable=invalid-name
        """UUIDs to include in "Service Solicitation" Advertisement Data"""
        return self.SolicitUUIDs.unpack()

    @solicit_UUIDs.setter
    def solicit_UUIDs(self, data: List[str]) -> None:
        self.SolicitUUIDs = GLib.Variant('as', data)

    @property
    def service_data(self):
        """Service Data to be broadcast"""
        return self.ServiceData.unpack()

    @service_data.setter
    def service_data(self, service_data):
        s_data = {}
        for key, value in service_data.items():
            gvalue = GLib.Variant('ay', value)
            s_data[key] = gvalue
        self.ServiceData = s_data

    @property
    def local_name(self) -> Union[str, None]:
        """Local name of the device included in Advertisement."""
        if self.LocalName:
            return self.LocalName.unpack()
        return None

    @local_name.setter
    def local_name(self, name: Union[str, None]):
        if name:
            self.LocalName = GLib.Variant.new_string(name)
        else:
            self.LocalName = None

    @property
    def appearance(self) -> int:
        """Appearance to be used in the advertising report."""
        return self.Appearance

    @appearance.setter
    def appearance(self, appearance: int) -> None:
        if appearance:
            self.Appearance = GLib.Variant.new_uint16(appearance)
        else:
            self.Appearance = None


def main():
    # Simple test
    beacon = Advertisement(1, 'peripheral')
    beacon.service_UUIDs = ['FEAA']
    beacon.service_data = {'FEAA': [0x10, 0x08, 0x03, 0x75, 0x6B,
                                    0x42, 0x61, 0x7A, 0x2e, 0x67,
                                    0x69, 0x74, 0x68, 0x75, 0x62,
                                    0x2E, 0x69, 0x6F]}
    beacon.start()
    ad_manager = bluez_proxy(ADAPTER_PATH, LE_ADVERTISING_MANAGER_IFACE)
    ad_manager.RegisterAdvertisement('(oa{sv})', beacon.path, {})
    mainloop = GLib.MainLoop()
    mainloop.run()


if __name__ == '__main__':
    main()
```
### Debugging Issues

Because of the complexity of the data structures it is not uncommon for
BlueZ to throw and error saying in cannot parse the advertisement data.
A practical way to debug this is remove the `RegisterAdvertisement` step
and access the service that has been published.
This can be done on the command line using `busctl list` to see the new 
service that has successfully been published to the system D-Bus.
If this is a long list then using `busctl list | grep python` will help
reduce the list.
The published service will have a unique name (i.e. :x.xxx) that is not
very human-readable. This is done to allow the creation of the service 
without having to change security settings on the system bus. 
You can now use `busctl tree :x.xxx` and other `busctl` commands to interact
with the service you created and published. 

---

&copy; Copyright 2022, Barry Byford.

first published: 2022 February 6

last updated: 2022 February 6

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
