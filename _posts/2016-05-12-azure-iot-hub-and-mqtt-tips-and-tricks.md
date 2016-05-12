---
layout: post
date: 2016-05-12
title: Azure IoT Hub and MQTT tips & tricks
---

Azure IoT Hub has had MQTT support for a little while now, as documented in this article: [IoT Hub MQTT support](https://azure.microsoft.com/en-us/documentation/articles/iot-hub-mqtt-support/). However it might sometimes be tricky to figure out how to use this information with currently available MQTT libraries. Here are a couple tips & tricks and code samples to get you started.

First, read carefully the section titled [Using the MQTT protocol directly](https://azure.microsoft.com/en-us/documentation/articles/iot-hub-mqtt-support/#using-the-mqtt-protocol-directly); it describes how to assemble the credentials required to connect to the device endpoints. The article is self explanatory, but here are some tips:

- You need to generate a SAS Token for the device to use as a password. You can do this on Windows as described in the article, using the DeviceExplorer GUI tool. You currently can't do it via the `iothub-explorer` command-line tool, but hopefully once [this PR](https://github.com/Azure/azure-iot-sdks/pull/551) is merged in, you should be able to use the new `sas-token` option.
- The article doesn't mention it, but you need to connect via SSL/TLS, using port 8883.
- Click through to the [Notes on MQTT support](https://azure.microsoft.com/en-us/documentation/articles/iot-hub-devguide/#mqtt-support) for another useful piece of information: IoT Hub implements the MQTT v3.1.1 protocol specifically.

Now let's looke at a simple example showing how to use the [Paho MQTT Python client](https://eclipse.org/paho/clients/python/) to send data to Azure IoT Hub. You need to assemble the rights credentials and configure TLS and the MQTT protocol version appropriately.

~~~ python
#!/usr/bin/python

import paho.mqtt.publish as publish
import paho.mqtt.client as mqtt
import ssl

auth = {
  'username':"ciscohackhub.azure-devices.net/lora1",
  'password':"SharedAccessSignature sr=ciscohackhub.azure-devices.net%2Fdevices%2Flora1&sig=xxxx&se=1463048772"
}

tls = {
  'ca_certs':"/etc/ssl/certs/ca-certificates.crt",
  'tls_version':ssl.PROTOCOL_TLSv1
}

publish.single("devices/lora1/messages/events/",
  payload="hello world",
  hostname="ciscohackhub.azure-devices.net",
  client_id="lora1",
  auth=auth,
  tls=tls,
  port=8883,
  protocol=mqtt.MQTTv311)
~~~

The trick here is to specify all the protocol versions (TLSv1, MQTTv311) and properly configure the CA certificates for TLS.

Here is another quick example, this time using the Node.JS [MQTT.js](https://github.com/mqttjs/MQTT.js) built-in command line tool "mqtt":

~~~ sh
mqtt publish -C mqtts \
-h ciscohackhub.azure-devices.net -p 8883 \
-i lora1 \
-u 'ciscohackhub.azure-devices.net/lora1' \
-P 'SharedAccessSignature sr=ciscohackhub.azure-devices.net%2fdevices%2flora1&sig=xxxx&se=1470823180' \
-t 'devices/lora1/messages/events' \
-m "hello mqtt"
~~~
