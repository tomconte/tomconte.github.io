---
layout: post
title: Deploying a private Ethereum blockchain on Azure and a Raspberry Pi
---

As part of exploring the blockchain technology, and specifically the Ethereum ecosystem, I have settled on an IoT-related use case around solar energy grids. You can read more background in the [first article in this series](/2016/05/26/exploring-the-blockchain-with-iot.html).

In this post, I am going to focus on building a small private blockchain that I will later use to deploy Smart Contracts and build my solar energy marketplace demo on. I am going to use an [Azure virtual machine](https://azure.microsoft.com/en-us/services/virtual-machines/) to start a reasonably powerful mining node, and a [Raspberry Pi 3](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) to simulate an on-premises equipment running a lighter, non-mining node, but which can still be involved in blockchain transactions.

We will be using [geth](https://github.com/ethereum/go-ethereum/wiki/Geth), the **Go Ethereum client**, to set up our cluster. Since the blockchain is basically a peer-to-peer network, geth is used both as a client and a server connecting to the blockchain. The installation procedures are fairly straightforward, I will be going through all the steps in the next few paragraphs.

## Installation on the Azure VM

You can start from one of the [Azure DevTest Labs](https://github.com/Azure/azure-blockchain-projects/tree/master/baas-artifacts) VM templates for Ethereum-Go, or you can install it from scratch on a base Ubuntu VM. In this documentation I will be starting from a blank VM, just because I want to understand how all the bits and pieces fit together. 

You can follow the [official Ethereum installation instructions](https://github.com/ethereum/go-ethereum/wiki/Installation-Instructions-for-Ubuntu) to install geth on your Ubuntu VM. I would recommend using the latest stable version, i.e. do not add the `ethereum-dev` repository.

~~~ shell
sudo apt-get install software-properties-common
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt-get update
sudo apt-get install ethereum
~~~

You can choose pretty much any [size of Azure VM](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-windows-sizes/) you would like, but since it is going to be mining (i.e. processing transactions) you probably want to go for a D2- or D3-class machine.

Once you have installed geth on the VM, we are ready to get started. First, to initialize a new private blockchain, we will need what is called a Genesis block, which is going to be the first block in our blockchain. This first block sets a few fundamental parameters for our blockchain, captured in a genesis.json file:

~~~ json
{
    "nonce": "0x0000000000000042",
    "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "difficulty": "0x4000",
    "alloc": {},
    "coinbase": "0x0000000000000000000000000000000000000000",
    "timestamp": "0x00",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "extraData": "Custom Ethereum Genesis Block",
    "gasLimit": "0xffffffff"
}
~~~

Most of these values are pretty much zeroes, except for `difficulty` which sets the mining difficulty to a low level so that we don't have to wait too long, or spent too much CPU time, waiting for transactions to be executed. You can read more about the Genesis block in [Ethereum's documentation for private test networks](http://ethdocs.org/en/latest/network/test-networks.html#the-genesis-file).

Copy/paste this bit of JSON in a file, then we can initialize our blockchain:

~~~ shell
geth init genesis.json
~~~

Once this is done, we are ready to run the geth console. The console is a JavaScript runtime environment that exposes Ethereum's JavaScript APIs and allows you to interact with the blockchain from the command line.

When running geth, we use a specific Network ID to differentiate from the public blockchain (whose Network ID is 1), and the `nodiscover` and `maxpeers` options to limit network connections for now. The goal is to make sure that we do not connect to the public blockchain by mistake.

~~~ shell
geth --networkid 42 --nodiscover --maxpeers 0 console
~~~

geth will complain that it has no default account, this is something we need to fix so we can start mining some Ether.

~~~
WARNING: No etherbase set and no accounts found as default
~~~

You can close the console using the `exit` command. Then create a new account:

~~~ shell
geth account new
~~~

Enter a password and don't forget it! You will need it to unlock that account. Geth will display your new account address:

~~~ shell
Address: {44878b3140a4e8072de0d1fbdec1ca58186fa17d}
~~~

Another way to do this, and also get familiar with the Ethereum JavaScript APIs, is to use the console directly:

~~~ javascript
personal.newAccount()
~~~

Once you have created an account, you can explorer the Ethereum APIs from the console, for example you can list existing accounts using `eth.accounts`, and you can see that our new account has been set as the "coinbase", i.e. the account that will receive mining rewards, using `eth.coinbase`.

Now we can start mining! The mining process is very verbose by default, so I suggest you use the `tmux` or `screen` commands to be able to open a separate, clean window to enter commands in the console.

~~~ shell
geth --mine --networkid 42 --nodiscover --maxpeers 0 console
~~~

The first time, geth will spend some time generating its [Ethash DAG](https://github.com/ethereum/wiki/wiki/Ethash); when it is finished, it will start mining blocks. While it is doing that, you can open a new virtual terminal using screen/tmux and attach a new geth console to the local node:

~~~ shell
geth attach
~~~

Then you can check your account balance from the console:

~~~ javascript
eth.getBalance(eth.accounts[0])
~~~

After a while, the mining process will start and your account will start getting credited with Ether. Of course this is not "real" Ether, this is just our own private Ether currency!

## Gathering network information

Now, let's build up our network a little bit by adding the Raspberry Pi as a peer node. In order to simplify things for this test/demo setup, I am going to configure direct connections, rather than using the Ethereum discovery protocol. In a larger private blockchain network, we would probably use bootstrap nodes in order to facilitate the discovery and connectivity of the different nodes.

First, we need to retrieve the enode address of our VM, so the node running on the Pi can connect to it.

From the console, type `admin.nodeInfo.enode` and you will something like this:

~~~ json
"enode://xxxxeda0xxxx66fcxxxx5d83xxxx35aaxxxx3787xxxx02e3xxxxdcc3xxxxaa3bxxxxac6exxxx9c2axxxxa543xxxx724d9b7eb02263e37ff5d7cba09ab63f7d5b@[::]:30303?discport=0"
~~~

You will use this address later to configure the Pi client. 

In order to allow clients to connect, you will also need to restart the geth node with a non-zero maxpeers parameter, e.g. :

~~~ shell
geth --mine --networkid 42 --nodiscover --maxpeers 3 console
~~~

## Installation on the Pi

I did not find a pre-built version of geth for the Raspberry, but it is easy to build it from the source code.

First, download and install [Go 1.6 from the official builds](https://golang.org/dl/). At the time of writing, the latest ARM build was:

~~~ shell
wget https://storage.googleapis.com/golang/go1.6.2.linux-armv6l.tar.gz
~~~

Compile geth on the Pi from the GitHub repo:

~~~ shell
git clone https://github.com/ethereum/go-ethereum
cd go-ethereum/
make geth
~~~

You will then need to use the same genesis file as above to initialize the node:

~~~ shell
geth init genesis.json
~~~

In order to connect this new node to our VM, create a file `static-nodes.json` to point our client to the Azure VM's public IP address. Just use the enode string we got from `admin.nodeInfo`, but replace the [::] with the Virtual Machine public IP address that you can find on the Azure Management Portal.

~~~ json
[
"enode://xxxx81d4xxxx7fc0xxxx00e1xxxx778cxxxx167dxxxx05e8xxxx2f2bxxxx1ddaxxxx504cxxxxd3ecxxxxb5e0xxxx6fce0038a9d6c0b9868c96ddb89877cf88e1@65.52.148.174:30303"
]
~~~

Then copy this file in place in the Ethereum data directory:

~~~ shell
cp static-nodes.json ~/.ethereum/
~~~

You can then run the node using the same `networkid` we used before; you can raise the verbosity just to see the networking messages and check everything is going OK.

~~~ shell
./go-ethereum/build/bin/geth --networkid 42 --nodiscover --verbosity 5 console
~~~

You should then see the synchronization start:

~~~
I0524 08:23:19.851867 eth/downloader/downloader.go:299] Block synchronisation started
~~~~ 

You can use the `admin.peers` command from the console to check the two nodes are connected. It can take a little while for the nodes to sync.

## Testing our private blockchain

On the Raspberry Pi, let'screate a new account, e.g. from the console:

~~~ javascript
personal.newAccount()
~~~

On the mining VM, you can send some Ether using the account you created previously: since this source account is receiving the mining rewards, it should now have a few Ethers to spend. First, unlock the account using the passphrase:

~~~ javascript
personal.unlockAccount('0x4487xxxx40a4xxxx2de0xxxxdec1xxxx186fxxxx')
~~~

Then send a transaction using the account addresses you created:

~~~ javascript
eth.sendTransaction({from: '0x4487xxxx40a4xxxx2de0xxxxdec1xxxx186fxxxx', to: '0x9a0dxxxx91cexxxx3034xxxx1800xxxx0b86xxxx', value: web3.toWei(1, "ether")})
~~~

The transaction should be received on the Pi, where you can get the balance account to check you received the Ether:

~~~ javascript
eth.getBalance(eth.accounts[0])
~~~

This is it, we now have a small, but working, **private blockchain network**! In the next few posts we will see how to write a Smart Contract, deploy it privately and then call it from different application contexts.

Any feedback is always welcome via Twitter: [@tomconte](https://twitter.com/tomconte).
