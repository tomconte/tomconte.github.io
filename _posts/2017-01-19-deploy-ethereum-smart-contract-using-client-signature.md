---
title: Compile and deploy an Ethereum smart contract using client-side signatures
layout: post
---

In my last post I showed you how to deploy an Ethereum Smart Contract programmatically, using the `web3` API. The code I showed works in the case you are working with a local Ethereum node, where you can safely unlock your account. However, if you are using a remote node, which is the case if you are using our [Bletchley infrastructure template](https://azure.microsoft.com/en-us/marketplace/partners/microsoft-azure-blockchain/azure-blockchain-serviceethereum-consortium-blockchain/) for example, then unlocking an account on the remote node exposes it to abuse, i.e. while it is unlocked, *anyone* can submit transactions on that account behalf. 

You could also enable and call the `personal.unlockAccount()` API on the node, however this also poses a big security risk: since the communication with the JSON-RPC endpoint is not protected using SSL, your credentials are exposed in clear text. 

In order to safely submit transactions to a remote node, you need to use a tool like [MetaMask](https://metamask.io/), which allows you to sign the transactions on the client side -- via the browser in the case of MetaMask. This works great in the [Online Solidity Compiler](https://ethereum.github.io/browser-solidity/) for example, but clicking through a Web app is not practical when you are a developer looking at streamlining your deployment process.

The code below should allow you to reproduce this process of "client-side transaction signing" in order to safely deploy Smart Contracts to a remote Ethereum node.

First we will need the usual Node.JS modules, with the addition of `ethereumjs-tx`, which is a utility library that we will use to sign our transactions.

``` javascript
const fs = require('fs');
const solc = require('solc');
const Web3 = require('web3');
const Tx = require('ethereumjs-tx')
```

Then we will need our private key in raw form, which is the first tricky part. Since this private key basically gives all the rights to your account, it is safely hidden in an encrypted form, inside your Ethereum node configuration directory. To find your private key file, look inside these directories:

- For `geth`: in `.ethereum/keystore`
- For Parity: in `.parity/keys`

Once you found your private key file, you can use a tool like [MyEtherWallet](https://www.myetherwallet.com/#view-wallet-info) to decrypt your private key and display it in RAW form: go to the Wallet Info tab, select "Keystore File", select your key file, and enter your password. The next page will display a bunch of details about your account, and you will need to copy and paste the "Private Key (unencrypted)" string in the variable below.

It's only OK to put this private key in the clear in the source code because we are dealing with a consortium/private blockchain account for development purposes; *do not do this with a real Ethereum public account!*

In this example, I used the private key file for the `web3.eth.coinbase` acount; if you are using a different account, you will need to change it below.

``` javascript
var privateKey = new Buffer('088c110b6861b6c6c57461370d661301b29a7570d59cb83c6b4f19ec4b47f642', 'hex')
```

Now we will connect to our remote node:

``` javascript
const web3 = new Web3(new Web3.providers.HttpProvider("http://tcoexownf.westeurope.cloudapp.azure.com:8545"));
```

And now we can compile our code as usual, and retrieve the contract data:

``` javascript
// Compile the source code
const input = fs.readFileSync('Token.sol');
const output = solc.compile(input.toString(), 1);
const bytecode = output.contracts['Token'].bytecode;
const abi = JSON.parse(output.contracts['Token'].interface);

// Contract object
const contract = web3.eth.contract(abi);

// Get contract data
const contractData = contract.new.getData({
    data: '0x' + bytecode
});
```

And now the interesting bit! We need to construct a raw transaction, including a few parameters we usually don't specify, like the gas and nonce to use:

``` javascript
const gasPrice = web3.eth.gasPrice;
const gasPriceHex = web3.toHex(gasPrice);
const gasLimitHex = web3.toHex(300000);

const nonce = web3.eth.getTransactionCount(web3.eth.coinbase);
const nonceHex = web3.toHex(nonce);

const rawTx = {
    nonce: nonceHex,
    gasPrice: gasPriceHex,
    gasLimit: gasLimitHex,
    data: contractData,
    from: web3.eth.coinbase
};
```

We can now sign this transaction using our private key:

``` javascript
const tx = new Tx(rawTx);
tx.sign(privateKey);
const serializedTx = tx.serialize();
```

And send it:

``` javascript
web3.eth.sendRawTransaction(serializedTx.toString('hex'), (err, hash) => {
    if (err) { console.log(err); return; }

    // Log the tx, you can explore status manually with eth.getTransaction()
    console.log('contract creation tx: ' + hash);

    // Wait for the transaction to be mined
    waitForTransactionReceipt(hash);
});
```

Now, since we are using a fairly low-level API, we are notified in the callback that the transaction has been accepted, but we also need to wait for it to be actually mined before our contract is available in the blockchain. There are several ways to do this, but I went for a simple loop that checks every second if the transaction was mined, by looking for a corresponding receipt:

``` javascript
function waitForTransactionReceipt(hash) {
    console.log('waiting for contract to be mined');
    const receipt = web3.eth.getTransactionReceipt(hash);
    // If no receipt, try again in 1s
    if (receipt == null) {
        setTimeout(() => {
            waitForTransactionReceipt(hash);
        }, 1000);
    } else {
        // The transaction was mined, we can retrieve the contract address
        console.log('contract address: ' + receipt.contractAddress);
        //testContract(receipt.contractAddress);
    }
}
```

Once you have retrieved the `contractAddress`, you can start interacting with the contract as usual.
