---
layout: post
date: 2015-03-09
title: Cross-compilation made easy on Ubuntu using sbuild
---

In my [previous post](/2015/03/06/cross-compiling-qpid-proton-on-ubuntu-arm.html), I showed you the manual steps to cross-compile the [Apache Qpid Proton](http://qpid.apache.org/proton/) libray to Ubuntu on ARM, with the goal of using the library to connect a Beaglebone device running Snappy Ubuntu Core to [Azure Event Hubs](http://azure.microsoft.com/en-us/services/event-hubs/) and other Azure services for IoT.

I have now a much improved method, thanks to excellent folks at Canonical who pointed it out to me!

The idea here is to use an Ubuntu developer tool called `sbuild`, which was designed to make it easy to cross-compile Ubuntu packages to other hardware architectures. In our case, we are not really going to cross-compile Ubuntu packages, but we can leverage this setup to make it much easier to cross-compile our libraries and programs.

Again, let's start with a vanilla Ubuntu 14.10 virtual machine.

First, let's install the tools:

```
sudo apt-get update
sudo apt-get install ubuntu-dev-tools
sudo apt-get git
```

Run the command to create the chroot:

```
mk-sbuild --arch=armhf vivid
```

This will install a bunch of other packages and ask you a couple of configuration questions. As instructed, when the installation is finished, log off and back on.

Run the same command again to **actually** create the chroot:

```
mk-sbuild --arch=armhf vivid
```

This will take a very very long time while all packages are installed! You can definitely grab a cup of coffee or two, even go for a walk around the block.

The chroot should get created in `/var/lib/schroot/chroots/vivid-armhf` where you will find all the system files, libraries, etc., for the ARM architecture.

You can now log into this chroot environment, it will behave like a virtual machine running on ARM, thanks to the magic of *qemu*!

```
sudo sbuild-shell vivid-armhf
```

You can now prepare this environment with all the prerequisites you need to compile Qpid Proton. Typically:

```
apt-get install python cmake git
apt-get install libssl-dev uuid-dev
```

Now let's see how to compile Qpid Proton in this virtual environment. The first thing you need is to get the source code in there! You should be able to work in this shared directory to do just that:

```
/var/lib/sbuild/build/
```

Do the following if you don't have the rights:

```
sg sbuild
```

And you should now be able to go to that directory and clone the Qpid Proton repo:

```
git clone -b 0.8 https://github.com/apache/qpid-proton.git
```

Now go back to the chroot using the shell command above, and you will find qpid-proton in `/build`, ready for you to build.

Now you can build as usual:

```
cd qpid-proton
mkdir build
cd build
cmake ..
make
make install
```

That's all folks! This method makes it really much easier to get all the right pre-requisites to work on a cross-platform project. In future posts we will see how to leverage this chroot environment to build your own apps.

Here are a few links I collected if you want to dig deeper into `sbuild`:

- [https://wiki.debian.org/sbuild](https://wiki.debian.org/sbuild)
- [https://wiki.ubuntu.com/SimpleSbuild](https://wiki.ubuntu.com/SimpleSbuild)
- [https://wiki.linaro.org/Platform/DevPlatform/CrossCompile/UsingMultiArch](https://wiki.linaro.org/Platform/DevPlatform/CrossCompile/UsingMultiArch)
