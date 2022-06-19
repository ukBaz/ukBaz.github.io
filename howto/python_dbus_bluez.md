# D-Bus and Bluez
## dbus-python

If you experiment with Python and Bluetooth on Linux then you will
possibly end up looking at the examples in the BlueZ source tree.
There are a number of examples such as:

https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/test/test-adapter

These examples use [dbus-python](https://dbus.freedesktop.org/doc/dbus-python/)
bindings which in the bindings own documentation suggest it might not be the
best  bindings to use:

> dbus-python might not be the best D-Bus binding for you to use. dbus-python 
> does not follow the principle of “In the face of ambiguity, refuse the 
> temptation to guess”, and can’t be changed to not do so without seriously 
> breaking compatibility.
> 
> In addition, it uses libdbus (which has known problems with 
> multi-threaded use) and attempts to be main-loop-agnostic (which means 
> you have to select a suitable main loop for your application).

This fact never makes me feel warn and friendly towards the BlueZ examples
but having done experiments with other bindings, there does not appear to be
a clear choice of bindings which will be usable for the full spectrum of 
BlueZ examples.

With my own Bluetooth libraries I've tried to avoid using those bindings.

For [ukBaz/BLE_GATT](https://github.com/ukBaz/BLE_GATT) it only covers the BLE
central role which meant I could use [pydbus](https://github.com/LEW21/pydbus).
These are very pythonic bindings which also allow development to be done in
a Python Virtual Environment (venv). 

The publishing of D-Bus services is not smooth with `pydbus`, and passing of
File Descriptors ar broken.

For my main Bluetooth library [ukBaz/python-bluezero](https://github.com/ukBaz/python-bluezero)
attempts to move to other bindings have always hit problems.

This blog is about an experiment to see if there was a way to do less verbose
code with the `dbus-python` bindings.

As with a lot of my experimentation this uses a BBC micro:bit as it has
some nicely 
[documented services](https://lancaster-university.github.io/microbit-docs/resources/bluetooth/bluetooth_profile.html) 
to interact with.

The experiment focused on three areas.
1) Building a Python proxy of the D-Bus object to interact with
2) Efficient way of find D-Bus path for GATT Characteristic's UUID
3) Convert D-Bus typed data to python types

### 1) Python Proxy Objects
`dbus.Interface` offers a nice way to access methods on a D-Bus interface
but not the properties. To access properties another Interface Proxy is
required for the same D-Bus object but using the 
`org.freedesktop.DBus.Properties` interface. This always felt clunky.

In this experiment I created the class `BluezProxy` which extended 
`dbus.proxies.Interface` with a couple of extra methods to simplify the
accessing of an interface's properties.

While the solution is not as Pythonic as would be ideal it does feel a 
positive step forward from the examples I've seen previously. The `get`
property is based on how 
[get](https://python-reference.readthedocs.io/en/latest/docs/dict/get.html)
is done in Python dictionaries.

### 2) Find D-Bus path for GATT Characteristic's UUID
`gatt_chrc_path` is the experiment to retrieve the GATT Characteristic path
from just knowing the UUID.
It is not wildly different from what has been done before but there are a 
couple of assumptions that have been adopted that are worth noting. 

`get_managed_objects` is the basis for actions like this as a way to get
the full information about what BlueZ has stored in D-Bus. Previously
this technique was used for finding all kinds of information which resulted
in complex functions. This experiment looked to see if all of them were 
necessary.

For Bluetooth Adapter there is usually only one on most machines and is
typically `/org/bluez/hci0`.

The remote device path is easy to derive if you know the Bluetooth address of
the remote device, and you only have one adapter. This is done with:
```python
DEVICE_PATH = f"{ADAPTER_PATH}/dev_{DEVICE_ADDR.replace(':', '_')}"
```

GATT service is often not necessary to know on something like the BBC micro:bit
which does not re-use characteristic UUIDs under different services.

This leaves the GATT Characteristic path as the one most commonly needing
to be found.
Focusing on just finding that path means the method can hard code 
the interface (`BLUEZ_GATT_CHRC`) and the property name (`UUID`).

If the UUID is available on multiple devices it there needs to ensure the path
is for the device of interest.

As D-Bus object paths are a tree like structure it allows for paths that 
don't start with the path of the required device to be filtered out.

The `object_path` is a Python property on the device proxy that was created
so can easily be accessed.

This does mean that creating a Python proxy of a GATT characteristic is a 
two-step process e.g:
```python
tmp_val_path = gatt_chrc_path(TMP_UUID, device.object_path)
tmp_chrc = BluezProxy(tmp_val_path, BLUEZ_GATT_CHRC)
```
which seems a reasonable trade-off for having `BluezProxy` as a more generic
function.

### 3) Convert D-Bus typed data to python types

A D-Bus `GetAll` properties on the adapter interface will by default give an
output like:
```python
{dbus.String('Address'): dbus.String('11:22:33:44:55:66', variant_level=1),
 dbus.String('AddressType'): dbus.String('public', variant_level=1),
 dbus.String('Alias'): dbus.String('TestMachine', variant_level=1),
 dbus.String('Class'): dbus.UInt32(786700, variant_level=1),
 dbus.String('Discoverable'): dbus.Boolean(False, variant_level=1),
 dbus.String('DiscoverableTimeout'): dbus.UInt32(190, variant_level=1),
 dbus.String('Discovering'): dbus.Boolean(False, variant_level=1),
 dbus.String('Modalias'): dbus.String('usb:v1D6Bp0246d0535', variant_level=1),
 dbus.String('Name'): dbus.String('TestMachine', variant_level=1),
 dbus.String('Pairable'): dbus.Boolean(False, variant_level=1),
 dbus.String('PairableTimeout'): dbus.UInt32(120, variant_level=1),
 dbus.String('Powered'): dbus.Boolean(True, variant_level=1),
 dbus.String('UUIDs'): dbus.Array([dbus.String('0000110e-0000-1000-8000-00805f9b34fb'),
                                   dbus.String('0000110a-0000-1000-8000-00805f9b34fb'),
                                   dbus.String('00001200-0000-1000-8000-00805f9b34fb'),
                                   dbus.String('00001112-0000-1000-8000-00805f9b34fb'),
                                   dbus.String('0000110b-0000-1000-8000-00805f9b34fb'),
                                   dbus.String('0000110c-0000-1000-8000-00805f9b34fb'),
                                   dbus.String('00001800-0000-1000-8000-00805f9b34fb'),
                                   dbus.String('00001108-0000-1000-8000-00805f9b34fb'),
                                   dbus.String('00001801-0000-1000-8000-00805f9b34fb')],
                                  signature=dbus.Signature('s'), variant_level=1)}
```
which is cluttered with D-Bus Type information. The `dbus_to_python`
functions allows for this to be changed to Python types and give the following
output:
```python
{'Address': 'FC:F8:AE:8F:0C:A4',
 'AddressType': 'public',
 'Alias': 'thinkabit1',
 'Class': 786700,
 'Discoverable': False,
 'DiscoverableTimeout': 190,
 'Discovering': False,
 'Modalias': 'usb:v1D6Bp0246d0535',
 'Name': 'thinkabit1',
 'Pairable': False,
 'PairableTimeout': 120,
 'Powered': True,
 'UUIDs': ['0000110e-0000-1000-8000-00805f9b34fb',
           '0000110a-0000-1000-8000-00805f9b34fb',
           '00001200-0000-1000-8000-00805f9b34fb',
           '00001112-0000-1000-8000-00805f9b34fb',
           '0000110b-0000-1000-8000-00805f9b34fb',
           '0000110c-0000-1000-8000-00805f9b34fb',
           '00001800-0000-1000-8000-00805f9b34fb',
           '00001108-0000-1000-8000-00805f9b34fb',
           '00001801-0000-1000-8000-00805f9b34fb']}
```

What `dbus_to_python` doesn't address is that Bluetooth sends data as a
series of bytes which may contain a physically grouped
list of variables under one read or write.

In the example below the `scroll delay` characteristic is a relatively simple
structure which only contains byte data for one variable of type `uint16`.
In python conversion to a single variable can be done with the
`int` or `struct` library. 

> **_NOTE:_** Bluetooth data is normally (always?) is `little-endian` format.
> 
```python
data = b'\x7e\x00'
value, = struct.unpack("<H", data)
print(value)  # 126
value = int.from_bytes(data, "little", signed=False)
print(value)  # 126
```
For more complex characteristics such as Heart Rate Measurement (not in the
example) which has multiple values in the one characteristic,
then using the `struct` library has a clearer advantage as it allows the 
multiple values to be unpacked at once. e.g:
```python
flags, hr_value, energy_expended = struct.unpack("<BBH", b"\x08\x40\xc8\x01")
# flags = 0b00001000; hr_value = 64; energy_expended = 456
```

The full result of the experiment is a relative short BLE client to 
interact with a BBC micro:bit.
```python
from pprint import pprint
from time import sleep
import dbus

BLUEZ_SERVICE_NAME = "org.bluez"
BLUEZ_ADAPTER = "org.bluez.Adapter1"
BLUEZ_DEVICE = "org.bluez.Device1"
BLUEZ_GATT_CHRC = "org.bluez.GattCharacteristic1"
TMP_UUID = "e95d9250-251d-470a-a062-fa1922dfa9a8"
DSPLY_TXT = "e95d93ee-251d-470a-a062-fa1922dfa9a8"
SCRLL_DLY = "e95d0d2d-251d-470a-a062-fa1922dfa9a8"
ADAPTER_PATH = "/org/bluez/hci0"
DEVICE_ADDR = "E1:4B:6C:22:56:F0"
DEVICE_PATH = f"{ADAPTER_PATH}/dev_{DEVICE_ADDR.replace(':', '_')}"


bus = dbus.SystemBus()


def dbus_to_python(data):
    """convert D-Bus data types to python data types"""
    if isinstance(data, dbus.String):
        data = str(data)
    elif isinstance(data, dbus.Boolean):
        data = bool(data)
    elif isinstance(data, dbus.Byte):
        data = int(data)
    elif isinstance(data, dbus.UInt16):
        data = int(data)
    elif isinstance(data, dbus.UInt32):
        data = int(data)
    elif isinstance(data, dbus.Int64):
        data = int(data)
    elif isinstance(data, dbus.Double):
        data = float(data)
    elif isinstance(data, dbus.ObjectPath):
        data = str(data)
    elif isinstance(data, dbus.Array):
        if data.signature == dbus.Signature('y'):
            data = bytearray(data)
        else:
            data = [dbus_to_python(value) for value in data]
    elif isinstance(data, dbus.Dictionary):
        new_data = dict()
        for key in data:
            new_data[dbus_to_python(key)] = dbus_to_python(data[key])
        data = new_data
    return data


def get_managed_objects():
    """
    Return the objects currently managed by the D-Bus Object Manager for BlueZ.
    """
    manager = dbus.Interface(
        bus.get_object(BLUEZ_SERVICE_NAME, "/"),
        "org.freedesktop.DBus.ObjectManager"
    )
    return manager.GetManagedObjects()


def gatt_chrc_path(uuid, path_start="/"):
    """
    Find the D-Bus path for a GATT Characteristic of given uuid.
    Use `path_start` to ensure it is on the correct device or service
    """
    for path, info in get_managed_objects().items():
        found_uuid = info.get(BLUEZ_GATT_CHRC, {}).get("UUID", "")
        if all((uuid.casefold() == found_uuid.casefold(),
                path.startswith(path_start))):
            return path
    return None


class BluezProxy(dbus.proxies.Interface):
    """
        A proxy to the remote Object. A ProxyObject is provided so functions
        can be called like normal Python objects.
     """
    def __init__(self, dbus_path, interface):
        self.dbus_object = bus.get_object(BLUEZ_SERVICE_NAME, dbus_path)
        self.prop_iface = dbus.Interface(self.dbus_object,
                                         dbus.PROPERTIES_IFACE)
        super().__init__(self.dbus_object, interface)

    def get_all(self):
        """Return all properties on Interface"""
        return dbus_to_python(self.prop_iface.GetAll(self.dbus_interface))

    def get(self, prop_name, default=None):
        """Access properties on the interface"""
        try:
            value = self.prop_iface.Get(self.dbus_interface, prop_name)
        except dbus.exceptions.DBusException:
            return default
        return dbus_to_python(value)


def main():
    """
    Procedurally connect to remote device and interact with various
    BLE GATT characteristics, and then disconnect from device
    """
    # dongle - Get information from Bluetooth dongle on device
    dongle = BluezProxy(ADAPTER_PATH, BLUEZ_ADAPTER)
    print(f"Discovery Filters: {dbus_to_python(dongle.GetDiscoveryFilters())}")
    print(f"Powered: {dongle.get('Powered')}")
    pprint(dongle.get_all())

    # Device - Connect to device with given address
    device = BluezProxy(DEVICE_PATH, BLUEZ_DEVICE)
    device.Connect()
    while not device.get("ServicesResolved"):
        sleep(0.5)

    # GATT - Read Temperature characteristic from device
    tmp_val_path = gatt_chrc_path(TMP_UUID, device.object_path)
    tmp_chrc = BluezProxy(tmp_val_path, BLUEZ_GATT_CHRC)
    value = int.from_bytes(bytes(tmp_chrc.ReadValue({})), "little")
    print(f"Temperature is: {value}")

    # GATT - Change Scroll Delay characteristic value
    scrll_dly_path = gatt_chrc_path(SCRLL_DLY, device.object_path)
    scrll_dly = BluezProxy(scrll_dly_path, BLUEZ_GATT_CHRC)
    print(f"Scroll speed [raw bytes]: {dbus_to_python(scrll_dly.ReadValue({}))}")
    value = int.from_bytes(bytes(scrll_dly.ReadValue({})), "little")
    print(f"Scroll delay is: {value}")
    scrll_dly.WriteValue(int(value + 2).to_bytes(2, "little"), {})
    value = int.from_bytes(bytes(scrll_dly.ReadValue({})), "little")
    print(f"Scroll delay is: {value}")

    # GATT - Write to Text characteristic
    txt_path = gatt_chrc_path(DSPLY_TXT, device.object_path)
    txt_chrc = BluezProxy(txt_path, BLUEZ_GATT_CHRC)
    txt_chrc.WriteValue(b'Carpe diem', {})
    # Disconnect from device
    device.Disconnect()


if __name__ == "__main__":
    main()

```
---

&copy; Copyright 2022, Barry Byford.

first published: 2022 June 19

last updated: 2022 June 19

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
