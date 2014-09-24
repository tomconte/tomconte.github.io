---
layout: post
date: 2014-09-24
title: Uploading files to Microsoft Azure Storage from the Tessel board using Node.JS
---

The latest gadget I have been playing with for a couple of days is the Tessel board by Technical Machine. I find it a great prototyping board for a few reasons: first it is downright tiny, smaller than an Arduino Uno, and it has four modules ports where you can plug additional modules -- like Arduino shields, except they are much smaller. I bought mine with an [Ambient](http://start.tessel.io/modules/ambient) module (sound + light sensors) and the [Camera](http://start.tessel.io/modules/camera) module.

![My brand new Tessel](/images/tessel/tessel_camera_ambient.jpg)

The other amazing thing with the Tessel is that it supports JavaScript! You read this right: you can develop for this tiny microcontroller using Node.JS. I find this pretty amazing, and you can [read more about the architecture of the system](http://blog.technical.io/post/98257815497/how-tessel-works-the-basics) on Technical Machine's blog. Basically, the Tessel runs an embedded [Lua](http://www.lua.org/about.html) VM, and the command-line interface bundles a JavaScript-to-Lua compiler called [Colony](https://github.com/tessel/colony-compiler) that is used to convert the code for execution on the microcontroller. Of course this conversion step can cause problems with more complex Node.JS projects, but for most simples cases, a script that runs on your workstation will run as-is on your Tessel, and that is pretty cool.

Of course the first thing I wanted to try is to somehow connect the Tessel to Azure! I will explore scenarios around the Event Hubs soon, but in the meantime I wanted to try something simple, like uploading a picture to Azure Blob Storage.

Of course before you can talk to the mighty Cloud, you need to get your Tessel connected. This is incredibly easy, because this tiny board already has a onboard Wi-Fi chip as well! All you need to do is to run a command like this to connect to your Wi-Fi network:

```
tessel wifi -n narnia -p mysecretpassphrase
```

So here is a little sample, based on the basic [Camera module sample](http://start.tessel.io/modules/camera). This script will take a picture and upload it to Azure Blob Storage, using a Shared Access Key to securely access the destination container.

The Shared Access Key is used in cases where you want to give read/write access to a subset of a Storage account: instead of storing your Storage Key in the Tessel device, which would give any attacker full access to your account, you can store a signed URI that only gives access to a single container. In the script below, this signed URI is stored in the `blob_sas` variable.

How do you generate this SAS URI? For this first test I cheated a little bit and used an excellent tool called [Azure Management Studio](http://www.cerebrata.com/products/azure-management-studio/introduction) by Cerebrata; it gives you a nice UI to generate the SAS URI. First you can create an Access Policy to specify that you want to give read/write access to a Blob Container, and set an expiry date if you want:

![Create Access Policy](/images/tessel/image3.png)

Then you can select an existing container and select "Generate Signed URL": this will open a wizard that first asks you which policy you want to apply:

![Create signed URL](/images/tessel/image1.png)

And will then create all the URI parameters for you:

![Create signed URL](/images/tessel/image2.png)

All you need to do is to copy/paste the Query String component in your script to retrieve the signature and other Shared Access Key components. It is certainly feasible to generate the SAS parameters from JavaScript, but I was just too impatient to get started!

Using this technique, uploading a file to Blob Storage is really easy, using the standard `https` module. You need to generate a [Put Blob](http://msdn.microsoft.com/en-us/library/azure/dd179451.aspx) REST call, which is basically an HTTP PUT with a couple of required headers.

You will find the full source code in the Gist below. To run it you will first need to install the Node module for the Tessel Camera:

```
npm install camera-vc0706
```

You will need to change the Azure parameters, i.e. `blob_host`, `blob_container`, and `blob_sas`. Then you should be able to run the script on the Tessel:

```
tessel run azurecam.js
```

Full source code:

<script src="https://gist.github.com/tomconte/20b4a2af3a64052165a9.js"></script>
