---
layout: post
date: 2014-07-10
title: Introduction to the Arduino series
---

I am starting a new series revolving around applications for connected devices, as part of my ongoing focus on Internet Of Things (IOT). The first thing I have tried is to connect various hardware prototyping boards to the cloud, in order to get an idea of the possibilities and challenges. I wanted to use a hardware platform that is common and easy to use, and like many others I picked [Arduino](http://www.arduino.cc/) boards to get started, plus my trusted [Raspberry Pi](http://www.raspberrypi.org/) – but the Pi is so powerful it's not even funny!

The Arduino catalog contains lots of different boards based on variously powerful micro-controllers plus standard analog/digital inputs and outputs. They also have lots of "shields" to easily add powerful modules such as Ethernet and WiFi network access. The [development tools](http://arduino.cc/en/Main/Software) are easy to use, and there are lots of technical advice and libraries around.

The Raspberry Pi is a different beast: it is more of a full-fledged computing device, ready to be connected to network, screen and keyboard. It runs a a full Linux distribution based on Debian called [Raspbian](http://www.raspbian.org/). This makes it very easy to connect it to various servers or back-ends – as long as you can find your way around a Linux system: you can use all the tools, libraries and packages available on Linux to connect to pretty much anything out of the box.

The situation is a little bit more complicated – and thus interesting – on an Arduino: the network modules are add-ons, the micro-controller doesn't run any OS at all, and you only get access to a limited set of libraries giving you just a subset of the APIs available on a regular computer. Adding anything requires [cross-compiling](http://playground.arduino.cc/Code/CmakeBuild) stuff to the target AVR architecture.

Yet connecting an Arduino to the network is perfectly feasible, and I would like to show you in this series a few of the experiments I have been working on to connect these fun little boards to various Azure services:

- Using an [Arduino Uno](http://arduino.cc/en/Main/arduinoBoardUno), the simplest board, and an Ethernet shield to send telemetry data to Azure Mobile Services.
- using an [Arduino Yun](http://arduino.cc/en/Main/arduinoBoardYun), a powerful hybrid Arduino/Linux board, to securely connect to Azure Service Bus via SSL.
- using an [Arduino Due](http://arduino.cc/en/Main/arduinoBoardDue), a more powerful board, to securely connect to an MQTT server running on a VM, using a simple but secure form of SSL: SSL-PSK.

Along the way I hope to show you various projects built on the Arduino and various other peripherals and connected things, like webcams, Philips Hue lights, etc.
