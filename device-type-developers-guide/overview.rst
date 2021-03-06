Overview
========

The SmartThings architecture provides a unique abstraction of devices
from their distinct capabilities and attributes in a way that allows
developers to build applications that are insulated from the specifics
of which device they are using. For example, there are lots of
wirelessly controllable “switches”. A switch is any device that can be
turned On or Off. 

When a SmartApp interacts with the virtual representation of a device,
it knows that the device supports certain actions based on its
capabilities. A device that has the "switch" capability must support
both the "on" and "off" actions. In this way, all switches are the same,
and it doesn't matter to the SmartApp what kind of switch is actually
involved. 

This virtual representation of the device is called a device handler, or SmartDevice.

.. note::

    This layer of abstraction is key to the successful function and flexibility of the SmartThings platform. Architecturally, device handlers are the bridge between generic capabilities and the device or protocol specific interface actually used to communicate with the device.

The diagram below depicts where device handlers sit in the
SmartThings architecture.

.. figure:: ../img/device-types/smartthings-architecture.png
   :alt: Smart Things Architecture


In the example shown above, the job of the device handler (that is
implementing the "switch" capability) is to parse incoming,
protocol-specific status messages from the device and turn them into
normalized "events". It is also responsible for accepting normalized
commands (such as "on" and "off") and turning those into the
protocol-specific commands that can be sent to the device to affect the
desired action.

For example, for a Z-Wave compatible on-off switch, the incoming status
messages used by the device to report an "on" or "off" state are as
shown below:

==============	=================================
Device Command	Protocol-Specific Command Message
==============	=================================
on				command: 2003, payload: FF
off				command: 2003, payload: 00
==============	=================================

Whereas the device status reported to the SmartThings platform for the
device is literally just a simple "on" or "off".

Similarly, when a SmartApp or the mobile app invoked an "on" or "off"
command for a switch device, the command that is sent to the device handler is just that simple: "on" or "off". The device handler must
turn that simple command into a protocol-specific message that can be
sent down to the device to affect the desired action.

The table below shows the actual Z-Wave commands that are sent to a
Z-Wave switch by the device handler.

==============	=================================
Device Command	Protocol-Specific Command Message

On				2001FF
Off				200100
==============	=================================

Core Concepts
-------------

To understand how device handlers work, a few core concepts need to be discussed.

Capabilities
~~~~~~~~~~~~

Capabilities are the interactions that a device allows. They provide an abstraction layer that allows SmartApps to work with devices based on the capabilities they support, and not be tied to a specific manufacturer or model. 

Consider the example of the "Switch" capability. There are many unique switch devices - in-wall switches, Z-Wave switches, ZigBee switches, etc. All these unique devices have a device handler, and the device handlers support the "Switch" capability. This allows SmartApps to only require a device that supports the "Switch" capability, and thus work with a variety of manufacturer and model-specific switches. 

This code illustrates how a SmartApp might interact with a device that supports the "Switch" capability:

.. code-block:: groovy

    preferences() {
        section("Control this switch"){
            input "theSwitch", "capability.switch", multiple: false 
        }
    }

    def someEventHandler(evt) {
        if (someCondition) {
            switch.on()
        } else {
            switch.off()
        }

        // logs either "switch is on" or "switch is off"
        log.debug "switch is ${switch.currentSwitch}"
    }

The above example illustrates how a SmartApp requests a device that supports the "Switch" capability. It can then work with the device knowing that it will support all the commands and attributes that the "Switch" capability supports.

There is a `reference
document <https://graph.api.smartthings.com/ide/doc/capabilities>`__ that outlines all the supported capabilities.

Commands and attributes deserve their own discussion - let's dive in.

Commands
~~~~~~~~

Commands are the actions that your device can do. For example, a switch can turn on or off, a lock can lock or unlock, and a valve can open or close. In the example above, we issue the "on" and "off" command on the switch by invoking the ``on()`` or ``off()`` methods.

Commands are implemented as methods on the device handler. When a device supports a capability, it is responsible for implementing all the supported command methods.

Attributes
~~~~~~~~~~

Attributes represent particular state values for your device. For example, the switch capability defines the attribute "switch", with possible values of "on" and "off". 

In the example above, we get the value of the "switch" attribute by using the "current<attributeName>" property (``currentSwitch``). 

Attribute values are set by creating events where the attribute name is the name of the event, and the attribute value is the value of the event. This is discussed more in the `Parse and Events documentation <parse.html#parse-events-and-attributes>`__

Like commands, when a device supports a capability, it is responsible for ensuring that all the capability's attributes are implemented.

Actuator and Sensor
~~~~~~~~~~~~~~~~~~~

If you look at the `Capabilities taxonomy <https://graph.api.smartthings.com/ide/doc/capabilities>`__, you'll notice two capabilities that have no attributes or commands - "Actuator" and "Sensor".

These capabilities are "marker" or "tagging" capabilities (if you're familiar with Java, think of the Cloneable interface - it defines no state or behavior). 

The "Actuator" capability defines that a device has commands. The "Sensor" capability defines that a device has attributes.

If you are writing a device handler, it is a best practice to support the "Actuator" capability if your device has commands, and the "Sensor" capability if it has attributes. This is why you'll see most device handlers supporting one of, or both, of these capabilities.

The reason for this is convention and forward-looking abilities - it can allow the SmartThings platform to interact with a variety of devices if they *do* something ("Actuator"), or if they report something ("Sensor").

Protocols
---------

SmartThings currently supports both the `Z-Wave <http://en.wikipedia.org/wiki/Z-Wave>`__ and `ZigBee <http://en.wikipedia.org/wiki/ZigBee>`__ wireless protocols. 

Since the device handler is responsible for communicating between the device and the SmartThings platform, it is usually necessary to understand and communicate in whatever protocol the device supports. This guide will discuss both Z-Wave and ZibBee protocols at a high level.