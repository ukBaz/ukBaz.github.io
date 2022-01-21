Creating a custom BlueZ Pairing Agent
=====================================

# Introduction
The Bluezero library is based on the Linux BlueZ stack mainly focused on 
Bluetooth Low Energy (BLE). 

From time to time I am asked why there is no support for device pairing in it.
There are a few reason for why this has never seemed worth the effort:

  1) The `bluetoothctl` tool from BlueZ does a pretty good job
  2) Creating a custom pairing agent is fiddly multiplied by the amount of 
     different scenarios
  3) As pairing is a one-time provisioning step,
     you don't need to do it very often
  4) Does automating the security process reduce the security?

As a reminder, the reason pairing is performed is to establish keys which can 
be used to encrypt a Bluetooth connection between two devices.

If you don't care about having a secure connection between the two devices
then turn off the need for pairing. Or if you are not worried about 
Machine-In-The-Middle (MITM) attacks during pairing and want minimal user
interaction then use "Just Works" pairing.

If you do want to have control over who has access to and believe that a 
custom pairng agent is the way to go then read on...

# Documentation and References
There is documentation on the Agent API which can be found at:

https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc/agent-api.txt

There is also a Python example/test available in the source tree: 

https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/test/simple-agent

However, it probably does a bad job of explaining the complexity to pairing.
There is also the [Bluetooth Core Specification](https://www.bluetooth.com/specifications/specs/core-specification/),
but again, not really for the application developer

The Bluetooth SIG have done a [blog series](https://www.bluetooth.com/blog/bluetooth-pairing-part-1-pairing-feature-exchange/)
on Pairing which has good information but not focused on BlueZ.

# Complexity

This section is going to talk about the different options and scenarios that
Bluetooth Pairing can cover. The aim is to simplify it, or at least put it in
the same terminology that BlueZ uses. It is difficult to test all the scenarios
so if you see something you don't agree with, then please let me know. This
document will hopefully evolve to become more accurate over time.

## Roles

There are the two ends of the Bluetooth link. One end will initiate the 
pairing process and one end will respond. What role (Initiator or Responder)
you expect the Agent to be will dictate which methods you need to implement.

## Security Manager

There have been various improvements to how pairing is done at a low-level
in the Bluetooth specification.

In BlueZ there is a property on the device to anticipate whether legacy or
simple pairing will occur if pairing is initiated. If the property is set
to true, the device will only support the legacy (pre-2.1) pairing mechanism.

## IO Capabilities

Different Bluetooth devices have different input and output capabilities.
For example, a peripheral might have no display and only a couple of buttons.

| BlueZ Capability | Device Input Capacity | Device Output Capability |
|------------------|:---------------------:|:------------------------:|
| DisplayOnly      | None                  | Numerical | 
| DisplayYesNo     | Yes/No                | Numerical |
| KeyboardOnly     | Numerical             | None |
| NoInputNoOutput  | None                  | None |
| KeyboardDisplay  | Numerical             | Numerical |



## Out-Of-Bands Pairing

This is when an external means of communication is used to exchange some
information used in the pairing used. This NFC or QR Code for example.

---

&copy; Copyright 2021, Barry Byford.

last updated: 2021 December 24

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
