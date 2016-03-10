---
layout: post
date: 2016-02-15
title: Sending messages to Azure IoT Hub in Python using Qpid Proton
---

While our [Azure IoT SDK](https://github.com/Azure/azure-iot-sdks) team is hard at work on an official Python SDK, we have received some requests from customers who were eager getting started using the service using currently available solutions. Python is a pretty easy language/platform to use when prototyping IoT solutions, especially when using devices like the Raspberry Pi where Python is a first-class citizen with many SDKs covering its hardware capabilities.

Until recently, we were using the Apache Qpid Proton libraries under the covers to implement the AMQP communication channel with IoT Hub. We are currently phasing out this library in favor of our Open Source implementation, named [uAMQP](https://github.com/Azure/azure-uamqp-c). However it is still possible to use the Python bindings that are bundled with the Proton library to send messages to IoT Hub.

First you will need to build the Proton library using [my own forked version](https://github.com/tomconte/qpid-proton/tree/0.9-IoTClient). It is based on the [Azure IoT original fork](https://github.com/Azure/qpid-proton/tree/0.9-IoTClient), with just one tiny patch added to fix the parsing of the AMQP URL, which otherwise gets confused by the username format we need to use, i.e. containing an "at" sign (see below in the Python snippet).

If you have the right packages installed, the standard build process should generate the Python bindings in `/usr/lib/proton/bindings/python`. Also make sure you add this directory to your Python search path:

	$ export PYTHONPATH=/usr/lib/proton/bindings/python

Now you should be able to send messages using the Python code below. Note the format of the URL:

	amqps://<username>:<password>@<iothubname>.azure-devices.net:/devices/<devicename>/messages/events

Where:

- `iothubname` is the name of your IoT Hub
- `devicename` is the name, or ID, of your device
- `username` is in the form `<device>@sas.<iothubname>`
- `password` is a valid Shared Access Signature token for the device (generated using the Device Explorer or any other tool or code)

The fact that the URL finally contains two "at" signs (one for the user name and one for the standard URL authority syntax) is what confuses the original Proton URL parsing code.

Hope this helps!

~~~ python
#!/usr/bin/python

# Send messages to an Azure IoT Hub using the Apache Qpid Proton AMQP library.

import urllib
from proton import *

# IoT Hub address & credentials

# The Device ID
device='myFirstDevice'
# The name of your IoT Hub
hubname='tcontehub'
# The SAS for the device; generate it using e.g. Device Explorer or the iothub-explorer command line tool
sas='SharedAccessSignature sr=tcontehub.azure-devices.net%2fdevices%2fmyFirstDevice&sig=xxxx&se=1463330495'

# Assemble the URL parts
username=device + '@sas.' + hubname
host=hubname + '.azure-devices.net'
path='/devices/' + device + '/messages/events'

# The final URL
address = 'amqps://' + username + ':' + urllib.quote(sas) + '@' + host + path

print address

# Create Messenger object
messenger = Messenger()

# Create AMQP message
message = Message()
message.body = unicode("Hello World!")
message.address = address
messenger.put(message)
messenger.send()
~~~


