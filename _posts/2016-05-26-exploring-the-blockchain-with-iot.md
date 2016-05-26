---
published: false
layout: post
date: 2016-05-26T00:00:00.000Z
title: Exploring the Blockchain with IoT
---

The blockchain has become a hot topic lately, with customers and partners in all sectors scrambling to experiment how the technology could impact their business. I wanted to get my hands dirty with a little prototype that I could demo, and decided to combine new hype with the old with a use case that marries the blockchain with the Internet of Things.

The association is not that exotic, with quite a few companies experimenting in that area. For example, a green energy startup called [LO3](http://lo3energy.com/) has [recently started trading energy through an Ethereum-based "MicroGrid" in Brooklyn](http://www.coindesk.com/ethereum-used-first-paid-energy-trade-using-blockchain-technology/). And of course there is [Slock.it](https://slock.it/), one of the Ethereum ecosystem rock stars, who envisions a future where a smart lock on an hotel door will automatically open once you have paid for your stay (read more about [Slock.it and Azure](https://blog.slock.it/slock-it-working-with-microsoft-to-bring-its-dapp-to-the-azure-cloud-c7a39720fdb3)).

In my case, I decided to explore the "smart energy" theme after being inspired by the [SolarChange](http://www.solarchange.co/) project, where "green energy producers to get a reward for their contribution to the planet" through a currency called "SolarCoins". Basically, I would like to build a connected device that automatically sells excess solar energy produced by a solar panel installation, via a blockchain-based peer-to-peer platform. This would allow the installation owner to earn some credits (or "coins" or "tokens") that he would later be able to spend in some way, maybe to purchase additional energy from other users when he needs it.

In practical terms, I will use the most versatile of small computers, a Raspberry Pi, to simulates a device that manages solar panels: some excess energy will be measured (in watts) and sold (in kWh) or exchanged via a blockchain using a custom currency. I am far from being a solar energy expert, but the basic units I will be dealing with should be something like this:

- One typical solar panel can produce 250 watts in peak conditions
- Thus four panels would produce 1 kW
- Electricity bills are expressed in kWh (average energy consumed over a period of time)
- 1 kWh = 1 kW sustained for one hour

So, if we imagine for example that I have left my home for a month-long vacation as we French people like to do, the Pi would automatically sell on the market all the unused energy produced by my four-panel solar installation, e.g. a total of 24 * 30 = 720 kWh. The Pi could for example issue a transaction for every excess kWh produced.

Of course I will also need a fancy cool name for my custom currency, I haven't really settled on one yet but since this is a sun-related project, I am counting on mythology for inspiration: maybe SunCoins? ApolloCoins, or simply appollos? HelioCoins? Lugs? Ras? Sols? SolCoins? Lots of ideas!

## Architecture

Here are the key design points of the test/demo architecture I would like to build:

- I am going to use Ethereum to build the blockchain, since this is one of the most popular and promising application platforms at the moment.
- The blockchain will be private.
- The mining nodes will run in the cloud and the devices (Raspberry Pi) will only act as clients.
- Since we are running on a private blockchain, and not the public one, we don't have to deal with the size of the public Ethereum blockchain, and we don't have to worry about using the light client protocol or other optimizations.
- We will not use the standard discovery protocol, we will specify the nodes addresses manually (or maybe later, use some private bootstrap nodes).
- Some apps will run on the Pi and interact with the blockchain in different ways (background process, user interface) via the local Ethereum node.
- A public Web app will be able to display some general network statistics and information, interacting with the cloud-hosted node.

See you back here for the more technical posts! Any feedback is always welcome via Twitter: [@tomconte](twitter.com/share?text=%40tomconte%20).
