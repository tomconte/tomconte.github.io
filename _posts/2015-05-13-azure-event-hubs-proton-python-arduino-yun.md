---
layout: post
date: 2015-05-13
title: Azure Event Hubs messaging using AMQP with Qpid Proton and Python from the Arduino Yun
---

Now **that** is a mouthful of a title! I have been slowly working on getting the Apache Qpid Proton library working on the Arduino Yun so that I can send messages to Event Hubs using AMQP instead of HTTP. It took a bit more work than expected, but as described in a previous blog post, I managed to [cross-compile the library to run on the Arduino Yun](http://hypernephelist.com/2015/03/30/qpid-proton-for-the-arduino-yun.html) using the OpenWRT toolchain. I also extended the procedure to [add the Python bindings in there](http://hypernephelist.com/2015/04/07/compiling-python-bindings-for-the-arduino-yun.html) to make the library easier to use; as described in the article, you will also need a Python shared library in order to be able to use these bindings.

Using Proton from a C program is actually fairly simple as well, and we have [some sample C code available](https://github.com/Azure/azure-content/blob/master/includes/service-bus-event-hubs-get-started-send-c.md), however from a practical perspective this would require you to cross-compile this C code to the Yun's MIPS environment and that is rather painful. Using Python makes it much easier to interface with Event Hubs from an Arduino sketch.

Here are some hopefully simple steps to install everything on your Yun and start using Proton!

First, here here are convenient download links to get the prebuilt binaries:

- [openwrt-qpid-proton_0.8-1_ar71xx.ipk](https://dl.dropboxusercontent.com/u/2264910/proton-yun/openwrt-qpid-proton_0.8-1_ar71xx.ipk)
- [libpython2.7.so.1.0](https://dl.dropboxusercontent.com/u/2264910/proton-yun/libpython2.7.so.1.0)

You need to download these files, then copy them to your Arduino Yun, e.g. using the `scp` command or Putty's `pscp.exe`:

~~~
scp openwrt-qpid-proton_0.8-1_ar71xx.ipk libpython2.7.so.1.0 root@192.168.240.1:
~~~

Just replace the IP address by the one of your Yun. The default password for `root` on the Yun is `arduino`.

Now log on to your Yun via `ssh` to install the package and move the library in place:

~~~
opkg install openwrt-qpid-proton_0.8-1_ar71xx.ipk
mv libpython2.7.so.1.0 /usr/lib
~~~

This will install the Qpid Proton library and the Python bindings in `/usr/lib/proton`, and the Python library in `/usr/lib`.

Before you can use the Python bindings, you need to update `/etc/profile` to add an environment variable pointing to the Proton directory:

~~~
echo "export PYTHONPATH=/usr/lib/proton" >> /etc/profile
~~~

Then reboot your Yun to make sure that setting is taken into account.

Now you can use the following simple Python script to start sending data to Event Hubs ([Gist](https://gist.github.com/tomconte/c94af7efeccb2a274b82)). You will need to update the AMQP address for your event hub, including the **URL-encoded key**! The comments hopefully describe the different components clearly, they can all be found in the Management Portal under your Event Hubs configuration tab.

~~~python
#!/usr/bin/python

# Send messages to an Azure Event Hub using the Apache Qpid Proton AMQP library.

import sys
import commands
from proton import *

# Event Hub address & credentials
# amqps://<keyname>:<key>@<namespace>.servicebus.windows.net/<eventhubname>
# You can find <keyname> and <key> in your Service Bus connection information
# <key> needs to be URL-encoded
# <namespace> is your SB top-level namespace
# <eventhubname> is the name of your Event Hub

address = "amqps://send:xxxx@demohub.servicebus.windows.net/yun";

# Create Proton objects
messenger = Messenger()

# Create AMQP message
message = Message()

# Add a body
message.body = unicode(sys.argv[1])

# Set the address and send the message
message.address = address
messenger.put(message)
messenger.send()
~~~

As you can see, the Python code is very simple! It takes the first parameter passed on the command line and puts it in the body of an AMQP message that is then sent to your Event Hub.

Run the script like this:

~~~
chmod +x event_hubs_send.py
./event_hubs_send.py "hello from yun"
~~~

You should see the new message show up in your Event Hub. For an easy way to check what is going on in Event Hubs, use [Service Bus Explorer](http://aka.ms/sbexplorer) on Windows, which lets you consume the messages. For Linux / Mac OS, I have a small Python script you can run to dump all messages from an Event Hub.

Now that this script is installed and working from the Yun command line, we can call it from a sketch running on the microcontroller, using the `Process` library!

~~~c
#include <Process.h>

void setup() {
  Bridge.begin();
  Serial.begin(9600);
}

void loop() {
  // Read a value
  int sensorVal = analogRead(A4);
  
  String data = String(sensorVal);
  
  Serial.println("Sending " + data);
  
  Process p;
  p.runShellCommand("/root/event_hubs_send.py " + data);
  
  delay(1000);
}
~~~

Of course you could improve the script and sketch to handle error codes, etc.

In summary I think this is a reasonably simple use of the Proton library on the Arduino Yun using Python. Hope it helps!
