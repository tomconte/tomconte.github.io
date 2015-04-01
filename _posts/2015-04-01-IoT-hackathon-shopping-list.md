---
layout: post
date: 2015-04-01
title: An IoT Hackathon shopping list
---

We have been running a few IoT hackathons here in Europe, as part of our role evangelising Microsoft Azure services for connected devices. One of our goals during these hackathons was to prove that you could use our Azure services for IoT from pretty much any connected device, even if it was not Microsoft-based in any way; of course most people expect that you can use Azure services from a .NET Micro Framework-based device like [Netduino](http://www.netduino.com/) and [Gadgeteer](https://www.ghielectronics.com/technologies/gadgeteer), but a lot of people already use Arduino and Raspberry Pi and Tessel devices, and we wanted to show that you could use Azure services from those too!

So we tried a bunch of different device types and we typically offer two or three different options during the hackathons. People can choose the device they prefer depending on what they are trying to achieve.

This shopping list summarizes the different options we considered and used, hoping it will be useful feedback for anyone planning an IoT hackathon.

# Devices & Sensors

## Tessel

The **Tessel** is the easiest device to use, but also the most expensive ($75). It has onboard Wi-Fi and easily pluggable sensor modules. It can be programmed in JavaScript and we have extensive code samples to connect it to Azure.

The Tessel comes with its Micro-USB cable and everything else is integrated, so you only need to buy some sensor modules. You can get nice modules for $25 apiece, such as the Ambient, Climate or Accelerometer modules. They also have bundles in the store with several Tessels and a selection of modules.

Tessel just announced a new Tessel 2 model that might be more powerful but less expensive; check it out, currently on pre-order!

Buy it directly on [https://tessel.io/](https://shop.tessel.io/) (international shipping from the US)

## Arduino Yun

There are many flavors of Arduino boards. We selected the **Arduino Yun** ($60) because it has onboard network including Ethernet and Wi-Fi, and is powerful enough (via its integrated Linux gateway) to connect directly to secure Azure services such as Event Hubs.

You can buy it directly from the [Arduino Store](http://store.arduino.cc/) but be sure to double-check inventory if you order many. Otherwise you can buy them from many online electronic stores.

The Arduino Yun does not include a Micro-USB cable by default, so you should add some to your list.

Regarding sensors, you basically have two strategies for the Arduino: either go with ready-made kits, or with basic electronic components.

### Ready-made sensor kits

This is the recommanded approach: these kits make it really simple to connect sensors and actuators to your Arduino boards without any knowledge in electronics. It is much easier for beginners than trying to use the electronic components directly.

The way the kits work is that they give you a base shield to plug on your Arduino. This shield has a series of proprietary connectors that let you connect various sensors in a foolproof way. This means that you need a base shield for every Arduino board.

Here are a couple of options:

**[TinkerKit](http://store.arduino.cc/category/16)** has a $80 base kit with one base shield and a dozen sensors. You can add additional base shields for $11 apiece.

**SeeedStudio** has the [Grove Starter Kit for Arduino](http://www.seeedstudio.com/depot/Grove-Starter-Kit-for-Arduino-p-1855.html) with a base shield and 10 sensors. Additional base shields $9 apiece.

### Electronic components

If you go for simple components like temperature sensors or photoresistors, they are not that hard to use. However you will need some breadboards, wires and resistors to build the circuits. The attendees will need some basic electronic knowledge, but no soldering.

For simple experiments, you can look for an "Arduino mini breadboard" that will fit neatly on top of the Yun.

For example on Amazon: **[mini breadboard with jumper wires](http://www.amazon.com/microtivity-170-point-Breadboard-Arduino-Jumper/dp/B004RXINEG/ref=sr_1_2?ie=UTF8&qid=1423814684&sr=8-2&keywords=arduino+mini+breadboard)**.

Look for packs of photoresistors and "TMP36" temperature sensors that are really easy to use.

Don't forget a **big pack of resistors** of different values, and some more **jumper wires**.

## Raspberry Pi

The Raspberry Pi or Pi 2 is more of a small really cheap ($50) PC than an embedded device. It can be used as a gateway for less powerful devices, for example. It also needs a lot of accessories to be really usable, e.g. power supply, a screen (HDMI), a keyboard, an SD card, and optionally a Wi-Fi dongle if you want wireless network access.

However it is also powerful enough to build nice projects around more powerful modules, like the Camera Module ($25).

It was also announced that the [Raspberry Pi 2 will run Windows 10](https://dev.windows.com/en-us/featured/raspberrypi2support), which makes it a great cross-platform device.

The best is to find a ready-made kit that includes typically:

- The Raspberry Pi board
- A Micro-USB power supply
- An SD Card with preinstalled OS
- A Wi-Fi USB dongle

# Network

If you go for Raspberry Pi devices, be aware that they don't have Wi-Fi by default. If you added a Wi-Fi dongle to the Pi kit you are good, but otherwise you will need Ethernet access. In all cases it is recommended to have a spare Ethernet switch in case the Wi-Fi dies or gets saturated.

# Sample list for 50 attendees

20 Tessel boards:

- Tessel board x 20
- Ambient module x 7
- Climate module x 7
- Accelerometer module x 6

20 Arduino Yun boards:

- Arduino Yun x 20
- Micro-USB cable x 20

20 Arduino Grove or Tinker Kits:

- Arduino sensor kit x 20

10 Raspberry Pi boards:

- Raspberry Pi x 10
- USB power supply x 10
- SD Card with OS x 10
- Wi-Fi USB dongle x 10 (optional if using Ethernet below)
- Raspberry Pi Camera Module x 10

Network:

- Ethernet switch, 16 ports (or more if necessary)
- Short Ethernet cables x 16

For setting up the Raspberry Pi boards:

- HDMI monitor (to share)
- HDMI cable
- USB keyboard
- USB mouse (optional)
