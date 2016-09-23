---
published: true
layout: post
title: Publishing a Smart Contract on the Azure Bletchley infrastructure
---

At the [Ethereum Devcon2](https://ethereumfoundation.org/devcon/) conference, Microsoft unveiled the first iteration of the infrastructure platform for the [Project Bletchley](https://azure.microsoft.com/en-us/blog/bletchley-blockchain/) vision, in the form of a [Quickstart ARM template](https://azure.microsoft.com/en-us/documentation/templates/ethereum-consortium-blockchain-network/) that can be used to easily deploy a fully configured blockchain cluster.

The template has great documentation, and we also have a detailed [walkthrough](https://aka.ms/blockchain-consortium-networks) to take you through all the steps, all the way to using the [MetaMask](https://metamask.io/) Ethereum wallet to send your first transactions.

I wanted to go one step further and deploy a Smart Contract using ConsenSys' [Truffle](https://truffle.readthedocs.io/en/latest/) tool/framework. I have been using Truffle on a few projects lately, and it is a great tool that facilitates compiling, deploying and using Smart Contracts.

Truffle is based on [Node.JS](https://nodejs.org/en/), so you will need to go and install that first; I usually go for the latest and greatest "Current" release.

On Windows, you will need a couple additional things, in order to get [node-gyp](https://github.com/nodejs/node-gyp) working so that you can build native modules: either the Visual C++ Build Tools or Visual Studio 2015, plus Python 2.7. You can also try Felix's [windows-build-tools](https://github.com/felixrieseberg/windows-build-tools) which should install everything in one go. Check out the [node-gyp README file](https://github.com/nodejs/node-gyp) for all the details!

Also make sure you install the very latest `npm` release:

```
npm install -g npm@next
```

Then you should be able to install Truffle:

```
npm install -g truffle
```

Truffle comes with a nice sample built in to get you started. Let's see how we can deploy this on our newly deployed Bletchley cluster!

First, create a new project:

```
md truffle-bletchley
cd truffle-bletchley
truffle init
```

This will scaffold a new distributed application for you, including the contracts.

On Windows, especially if you are using the `cmd.exe` command line, I suggest you rename the `truffle.js` configuration file to avoid any interactions with the interpreter:

```
ren truffle.js truffle-config.js
```

Feel free to explore the project: the Solidity contract, `MetaCoin.sol`, is in the `contracts` directory, and this is what we want to deploy! Typically as a developer, you will install and use [TestRPC](https://github.com/ethereumjs/testrpc) to test your contracts, and once it is stable and tested, you will want to deploy it onto your private blockchain.

In order to do that, Truffle has a concept of [networks](https://truffle.readthedocs.io/en/latest/advanced/networks/): basically, you can tell it the different places (i.e. blockchains) you want to deploy your contracts to. In order to target our Bletchley cluster, you will need to add a snippet like this one to `truffle-config.js`:

``` json
networks: {
  "bletchley": {
    network_id: 1337,
    host: "supersecret.northeurope.cloudapp.azure.com"
  }
}
```

Basically, you are giving Truffle the address of your HTTP-RPC endpoint (you can also add the port if it is different than 8545), and the network identifier of your private chain. Both can be found in the configuration of your Bletchley cluster.

Now you are ready to use Truffle's `migrate` command, which is used to compile and deploy the contracts, adding the `--network` option to specify our target:

```
truffle migrate --network bletchley
Running migration: 1_initial_migration.js
  Deploying Migrations...
Error encountered, bailing. Network state unknown. Review successful transactions manually.
Error: account is locked
```

Oops, what happened there? "account is locked"? Indeed, by default, for security purposes, neither Truffle nor `geth` give you any way to unlock your account remotely. In order to be able to deploy the contracts, and interact with the chain, you will first have to unlock the default account.

You can do this by using ssh to log on to your transaction (front-end) node, using the admin user and the host address, and then using `geth` to unlock the account:

```
ssh gethadmin@supersecret.northeurope.cloudapp.azure.com
```

Then attach to the running `geth` instance:

```
geth attach
```

And unlock your account:

```
personal.unlockAccount(eth.coinbase)
```

(enter the passphrase you used when you created the account)

You should now be able to exit `geth`, exit from the remote shell, and successfully deploy the contract:

```
> truffle migrate --reset --network bletchley
Running migration: 1_initial_migration.js
  Deploying Migrations...
  Migrations: 0xf0076d332f2a95d9750df5bbf2b9e45ced6802e3
Saving successful migration to network...
Saving artifacts...
Running migration: 2_deploy_contracts.js
  Deploying ConvertLib...
  ConvertLib: 0xd8f7eba541eb18e6854c7acada9d24fe862a1b15
  Linking ConvertLib to MetaCoin
  Deploying MetaCoin...
  MetaCoin: 0xd277628b4b911e8be1685a9cb48e15de160b5b4f
Saving successful migration to network...
Saving artifacts...
```

However, note that at the time of writing the template does not allow inbound SSH connections by default. You will need to add a NAT entry to the load-balancer configuration in order to allow these connections. You can do this using the Management Portal, by finding the Load Balancer component within your Bletchley Resource Group, then in the "Inbound NAT rules" section, adding a new rule for the SSH service, targeting your transaction VM node.

Truffle also has a neat testing console that will allow you to interact with your newly deployed contract. In the case of the MetaCoin example, you can try sending coins from your default account to another one (for example one you created using MetaMask, following the Bletchley walkthrough). This should work like this:

```
truffle console --network bletchley
```

Then you can grab a reference to your contract:

```
var c = MetaCoin.deployed()
```

And start using the contract functions. First check the balance of your default account; it should have 10,000 coins:

```
truffle(bletchley)> c.getBalance.call("0x708C77773a1c379aA70B0402Fa0dF12A9B00D76A")
{ [String: '10000'] s: 1, e: 4, c: [ 10000 ] }
```

Then try sending 100 coins to another account; this will trigger a transaction that will be mined on your private blockchain:

```
truffle(bletchley)> c.sendCoin("0x937b18971b8EFcB18DF492e7587636c2cDe50974", 100)
'0x75045124705f87743358856d1191c52ee7ca9918bf41637dba88cf1101c7dfad'
```

And finally check the coin balances of both accounts:

```
truffle(bletchley)> c.getBalance.call("0x708C77773a1c379aA70B0402Fa0dF12A9B00D76A")
{ [String: '9900'] s: 1, e: 3, c: [ 9900 ] }
truffle(bletchley)> c.getBalance.call("0x937b18971b8EFcB18DF492e7587636c2cDe50974")
{ [String: '100'] s: 1, e: 2, c: [ 100 ] }
```

This is it! You now have a working smart contract, deployed on your private Ethereum blockchain.
