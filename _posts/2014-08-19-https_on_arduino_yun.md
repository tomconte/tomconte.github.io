---
layout: post
date: 2014-08-19
title: Securing communications with HTTPS on the Arduino Yun
---

As I wrote in a previous post, a simple Arduino board like the Uno lacks the horsepower required to run an SSL stack that could be used to secure HTTP communications. However, Arduino has more sophisticated boards that can solve this problem. Today I would like to have a look at the [Arduino Yun](http://arduino.cc/en/Main/ArduinoBoardYun?from=Products.ArduinoYUN), a very sophisticated, yet tiny board.

The Yun really packs two different environments on a single board: an [ATmega32u4](http://www.atmel.com/dyn/resources/prod_documents/7766S.pdf) microcontroller, roughly of the same class as the one used on the Uno, plus an [Atheros AR9331](http://www.openhacks.com/uploadsproductos/ar9331_datasheet.pdf) processor running a Linux distribution based on OpenWRT. The Atheros processor is a Wi-Fi SOC typically used in Access Points and router platforms. This dual architecture allows you to keep running your low-level software on the microcontroller, and offload all the networking heavy lifting to the OpenWRT side of the board, where you have full support for pretty much any stack, protocol or utility you can run on the OpenWRT Linux derivative, like cURL, Python, and of course SSL in the form of OpenSSL or other lightweight stacks like PolarSSL.

The way both sides communicate is through a "Bridge", an Arduino library you can use in your sketch in order to run commands on the Linux side. The Bridge library also offers a host of other functions, like a network-based Console, File I/O to the integrated SD card, a specific wrapper for cURL, etc.

You should really read the [Yun guide](http://arduino.cc/en/Guide/ArduinoYun) to check out all the features, one of the great benefits of the Yun for programmers is that once it is online (through Wi-Fi or Ethernet) you can do everything remotely (i.e. without a USB cable), including uploading your code and accessing the Serial Monitor / Console.

In my case, I would like to explore some options to secure my HTTP calls. Since we have cURL already installed on the Yun and compiled with SSL support, it means we could send our data from the Linux command-line like this (note the _https_ protocol):

~~~
curl -k -v -X POST -H Content-Type:application/json \
-H X-ZUMO-APPLICATION:YaRWxxxxzKGRxxxxLPiNxxxxXYOvxxxx \
-d '{"test": 42}' \
https://arduinouno.azure-mobile.net/tables/telemetry
~~~

Here I am using `-k` to bypass certificate verification; ideally you would [install a CA cert bundle](http://curl.haxx.se/docs/sslcerts.html) to make sure the server is certificate is valid.

Now, all we need to do from our sketch is to run that same command, but using the Bridge library. This way we will move all networking code out of the sketch, freeing up resources on the microcontroller, and allowing us to secure the communications to the Azure Mobile Services back-end.

So, without further ado, here is the script modified for the Arduino Yun:

<script src="https://gist.github.com/tomconte/3ec8e8dae6296250e79e.js"></script>

As you can see, it is very similar to the Uno version, except we are using the `Bridge` library and especially the `Process` object, which allows us to run our command on the Linux side. The `Process` object can then be used to read the output of the command, in the case of cURL the response from the server. Below you can see the script running in the Arduino IDE:

![Running the script on the Yun](/images/yun_ams/Picture1.png)

In summary, the Yun gives us a simple way to emit SSL-protected HTTP requests. I will admit that running cURL for each outgoing request seems a bit overkill, but we could definitely explore other, more sophisticated options. One of them would be to use the [Yun package manager](http://arduino.cc/en/Tutorial/YunPackageManager) to install [stunnel](https://www.stunnel.org/static/stunnel.html), a very handy utility that can be used to create SSL tunnels. Once running on the Linux host, `stunnel` can forward plain-text HTTP connections (or any other protocol for that matter) over the encrypted tunnel.
