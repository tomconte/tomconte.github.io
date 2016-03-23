---
layout: post
date: 2016-03-21
title: A simple PHP library to communicate with Azure IoT Hub
---

A little while ago I published a post (in French) about [interfacing with the Jeedom home automation server]({% post_url 2016-03-15-interfacer-la-solution-domotique-jeedom-avec-azure-iot %}). But I wanted to be able to plug a bit deeper into the software, so I could collect all device events and send them to IoT Hub, without having to use any external tricks. Jeedom being written in PHP, that means I would need a PHP library that I can use to send telemetry data.

[Jeedom](https://www.jeedom.com/site/en/) is a nice piece of software that you can install on a small device like a Raspberry Pi and use as a controller for all your home devices like lights, all kinds of sensors, alarms, etc. I explored a few ways to interface Jeedom with [Azure IoT Hub](https://azure.microsoft.com/en-us/services/iot-hub/), mainly using Jeedom's existing features (the "Script" plugin) or external means like trolling its event logs. However, to get a really solid integration, I would need to add the IoT Hub calls directly in Jeedom's event management functions.

So I started working on a tiny PHP library, so that I would have the basic necessities to start sending data to IoT Hub using HTTP. I do not expect a lot of uptake since this is admittedly an edge case (I can't think of many devices that would run PHP!), but hopefully it can be useful to someone.

Currently the library is [hosted on my GitHub account](https://github.com/tomconte/azure-iot-hub), so in order to use it, your `composer.json` file should contain something like this:

~~~ json
{
    "repositories": [
        {
            "type": "vcs",
            "url": "https://github.com/tomconte/azure-iot-hub"
        }
    ],
    "require": {
        "tomconte/azure-iot-hub": "dev-master"
    }
}
~~~

This will load the Composer package directly from GitHub. Then you can use it that way:

~~~ php
<?php

require __DIR__ . '/vendor/autoload.php';

$host = 'hubbhub.azure-devices.net';
$deviceId = 'php_device';
$deviceKey = 'xxxxy5TAxxxx73VBxxxxq4WNxxxxaTIPxxxxEvkCxxxx';

$client = new AzureIoTHub\DeviceClient($host, $deviceId, $deviceKey);

$response = $client->send('Hello World!');

print($response->getStatusCode());
~~~

In order to find the values for `host`, `deviceId` and `deviceKey`, you can follow our [Get started with Azure IoT Hub](Get started with Azure IoT Hub) article. Basically, you will use the portal to create an IoT Hub, and then you will use our `iothub-explorer` command-line tool to generate a new device and its associated secret keys.

The `Send()` method will open an HTTPS connection and send a single message. There is currently no provision to reuse the connection, since in PHP, objects are typically short-lived.

Please note that as with all IoT Hub libraries, your secret key is never sent on the wire. The library will compute a Shared Access Signature (SAS) token and use it to authenticate the communication; you can look at the `computeSAS()` function in the source code to see how this is done.

Please use the GitHub repo if you use this little library and have any feedback, questions or issues! 
