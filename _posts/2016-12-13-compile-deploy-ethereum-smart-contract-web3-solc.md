---
title: Compile and deploy an Ethereum smart contract with solc and web3
layout: post
---

There are quite a few Ethereum development tools and environments out there, but sometimes it is useful to be able to deploy a smart contract programmatically using the simplest and most basic tools. For example, when testing various Ethereum implementations and variations, you might find that your development tool of choice doesn't interact well and is not able to deploy anything. This post is basically some notes about deploying a smart contract using simple, and hopefully universal, Node.JS code. 

First, we will need a small Solidity sample. I am using this minimal [Token contract](https://gist.github.com/tomconte/1946b121e69f8662e59613e7583e3ee3):

``` javascript
pragma solidity ^0.4.0;

contract Token {
    mapping (address => uint) public balances;

    function Token() {
        balances[msg.sender] = 1000000;
    }

    function transfer(address _to, uint _amount) {
        if (balances[msg.sender] < _amount) {
            throw;
        }

        balances[msg.sender] -= _amount;
        balances[_to] += _amount;
    }
}
```

You can save it under `Token.sol` if you want to test the code below.

Then we will move on to the actual work of reading the contract source code, compiling it, and deploying it to our development blockchain.

The code assumes that you have an Ethereum client running on your machine, and listening on the default JSON-RPC port (8545). For quick testing during development I would suggest [TestRPC](https://github.com/ethereumjs/testrpc), however this should work with any other "real" client, like [geth](https://github.com/ethereum/go-ethereum) or [parity](https://github.com/ethcore/parity), if they are suitably configured to run in a private development mode.

We will need a few Node.JS modules:

``` javascript
const fs = require('fs');
const solc = require('solc');
const Web3 = require('web3');
```

Then we need to connect to the local Ethereum node:

``` javascript
const web3 = new Web3(new Web3.providers.HttpProvider("http://localhost:8545"));
```

We are now going to read the whole source file in memory, and use the `solc` to compile it, and retrieve the two pieces of information we need to deploy the contract: its Application Binary Interface (ABI) and the bytecode produced by the compiler.

``` javascript
const input = fs.readFileSync('Token.sol');
const output = solc.compile(input.toString(), 1);
const bytecode = output.contracts['Token'].bytecode;
const abi = JSON.parse(output.contracts['Token'].interface);
```

Now we can create a contract object based on the ABI:

``` javascript
const contract = web3.eth.contract(abi);
```

Using this object, we can call the `new` method to deploy an instance of the contract. We need to pass the bytecode data, a `from` address (we will use the `coinbase` address, which we assume has lots of Ether), and some gas to pay for the deployment. In this example I just doubled the default value of 90000, but for a larger contract you might have to increase this value.

Note that the callback passed to the `contract.new` method is actually called twice: once when the transaction ID is known, and a second time when the contract has been deployed. In the second call, the `address` property will contain the address of the contract on the blockchain, which anyone call then use to interact with it.

``` javascript
const contractInstance = contract.new({
    data: '0x' + bytecode,
    from: web3.eth.coinbase,
    gas: 90000*2
}, (err, res) => {
    if (err) {
        console.log(err);
        return;
    }

    // Log the tx, you can explore status with eth.getTransaction()
    console.log(res.transactionHash);

    // If we have an address property, the contract was deployed
    if (res.address) {
        console.log('Contract address: ' + res.address);
        // Let's test the deployed contract
        testContract(res.address);
    }
});
```

As you can see at the end of the callback, we can now immediately test the contract to make sure it is working as expected, using the `testContract` method.

``` javascript
function testContract(address) {
    // Reference to the deployed contract
    const token = contract.at(address);
    // Destination account for test
    const dest_account = '0x002D61B362ead60A632c0e6B43fCff4A7a259285';

    // Assert initial account balance, should be 100000
    const balance1 = token.balances.call(web3.eth.coinbase);
    console.log(balance1 == 1000000);

    // Call the transfer function
    token.transfer(dest_account, 100, {from: web3.eth.coinbase}, (err, res) => {
        // Log transaction, in case you want to explore
        console.log('tx: ' + res);
        // Assert destination account balance, should be 100 
        const balance2 = token.balances.call(dest_account);
        console.log(balance2 == 100);
    });
}
```

This is just a simple test using the Web3 API:

- We first reference our newly deployed contract using the `contract.at()` function.
- Then the balance on the `coinbase` account is checked; according to the smart contract source code, it should contain 100000 tokens.
- Then the `transfer` function is called to send 100 tokens to another account.
- Once the transaction is executed, the balance on the receiving account is checked: it should of course contain 100 tokens.

Please note that this quick test is intended to be run on a development environment, where transactions are mined more or less instantly!

You can find the whole code as Gist here: [https://gist.github.com/tomconte/4edb83cf505f1e7faf172b9252fff9bf](https://gist.github.com/tomconte/4edb83cf505f1e7faf172b9252fff9bf).
