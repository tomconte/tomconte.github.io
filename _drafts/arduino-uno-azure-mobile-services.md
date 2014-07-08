---
layout: post
title: Connecting the Arduino Uno to Azure Mobile Services
---

In this post, I would like to show you how you can easily use [Azure Mobile Services](https://azure.microsoft.com/en-us/services/mobile-services/) to connect your [Arduino Uno](http://arduino.cc/en/Main/ArduinoBoardUno) device to the Cloud. Mobile Services, as its name implies, is intended to provide a scalable and secure backend for mobile applications running on any platform. While we do provide SDKs for all major platforms, we do not have something you can use out of the box from an Arduino. However, as I will show you in this article, you can start using Mobile Services from your Arduino with just a few lines of code.

I am starting with the [Uno](http://arduino.cc/en/Main/ArduinoBoardUno) because it is the entry-level prototyping board in the Arduino catalog, and I would like to see what we can do with this very minimal platform. It has 32KB of Flash memory to store your programs, but only 2KB of RAM, which is twice more than my good old [Sinclair ZX81](http://oldcomputers.net/zx81.html), but not enough to run any complex software! This means that we will have to keep our network access code very compact in order to leave some room for the "business logic" code you actually want to run on the micro-controller.

In order to connect to Azure from your Arduino board, you will of course need a network [shield](http://shieldlist.org/) of some sort. I will be using the official [Arduino Ethernet shield](http://arduino.cc/en/Main/ArduinoEthernetShield), but you should be able to use any compatible shield or a WiFi shield. As seen in the picture below, a shield piggybacks on top of your boad and allows you to access all the pins from the original board; here the Uno is seen with the quaintly artisanal wooden base and prototyping breadboard from the [Arduino Starter Kit](http://arduino.cc/en/Main/ArduinoStarterKit).

On the software side, we will use the standard [Arduino IDE](http://arduino.cc/en/main/software), which includes an [Ethernet library](http://arduino.cc/en/reference/ethernet) to easily connect to network resources. If you look in the example sketches bundled with the IDE, you will for example find Ethernet - WebClient which shows how to emit a simple HTTP GET request to a Web server and read the response. We don't need much more to connect to Azure Mobile Services!

## Creating a Mobile Service backend

Creating a Mobile Service backend couldn't be easier. Using the Web management portal, click on New, then Compute, Mobile Service, Create. You will need to choose a public URL for your service, a region where your service will be created, and the language you want to use: in this example I will use JavaScript. As part of the service, you are asked to create or connect to an Azure SQL Database that will be used to stored your data. If you don't have a database yet, you can opt to create a free 20MB database.

The next screen will ask you the details about your database: most importantly the server that will host it. You can choose an existing server if you have one, or create a new one on the fly (in this case you will have to select a login/password combination).




