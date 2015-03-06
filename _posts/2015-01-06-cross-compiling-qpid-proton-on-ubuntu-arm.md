---
layout: post
date: 2015-03-06
title: Cross-compiling Apache Qpid Proton for Ubuntu Snappy Core on ARM
---

I have recently been involved with Canonical when they launched their new ["Snappy" Ubuntu Core](http://www.ubuntu.com/things) operating system targeted at connected devices. The goal was to show how to connect a [Beaglebone](http://beagleboard.org/BLACK) device running this new distribution, connected to Microsoft Azure IoT services like **Event Hubs** and **Stream Analytics** to deliver real-time analytics. This demo ran on the Ubuntu booth at Mobile World Congress 2015.

The first step to connect the Beaglebone to Event Hubs was to compile the [Apache Qpid Proton](http://qpid.apache.org/proton/) library for the target ARM environment. Qpid Proton is a lightweight, portable AMQP 1.0 library that can be used to send messages to Event Hubs.

Fortunately, Qpid Proton has been designed with portability in mind, and its build system is based on CMake, which makes it reasonably easy to cross-compile the library from an *x86_64* box to the target *armhf* architecture. However, cross-compiling is more of black art than science, so after a few trial and errors here is how I got it to work "the hard way". I will hopefully explore some easier solutions provided by Ubuntu in future blog posts, based on creating a `chroot` environment containing all the necessary pre-requisites, instead of compiling everything by hand!

For now, let's see the manual way. I will start from scratch with a brand new Ubuntu 14.10 VM. My general strategy is to follow these steps:

- Cross-compile and install the packages in separate destination directories (e.g. `~/arm/library`)
- Copy the artifacts (headers, libraries) into the cross-platform library location (`/usr/arm-linux-gnueabihf`) so CMake can find them
- Run CMake pointing to this cross-platform library

## Installing cross-compiling tools for ARM target (Beaglebone)

First we need to install our usual build tools, plus the ARM toolchain (`gcc-arm-linux-gnueabihf`). We also need `git` and `cmake`.

```
sudo apt-get update
sudo apt-get install build-essential gcc-arm-linux-gnueabihf git cmake
```

## Cross-compiling qpid-proton dependencies

Qpid-proton has dependencies on OpenSSL and `libuuid`, so we need to compile those first.

### OpenSSL

The OpenSSL build seems slightly broken when cross-compiling, and there are a couple of manual steps to take.

First let's create our destination directory:

```
mkdir -p ~/arm/openssl
```

And install the OpenSSL source code:

```
wget https://www.openssl.org/source/openssl-1.0.2.tar.gz
tar xzf openssl-1.0.2.tar.gz
cd openssl-1.0.2
```

Let's create an environment variable that points to our target arhitecture:

```
export CROSS_COMPILE=arm-linux-gnueabihf-
```

Configure the OpenSSL package:

```
./Configure --prefix=~/arm/openssl shared -fPIC os/compiler:${CROSS_COMPILE}
```

Ignore the warning about shared libraries, we will fix that below :)

Now we need to fix a couple things that are broken in the generated Makefile.

Edit Makefile and fix the `CC` line:

```
CC= $(CROSS_COMPILE)gcc
```

Also modify the following lines so we can build the shared libraries:

```
SHLIB_TARGET=linux-shared
SHLIB_EXT=.so.$(SHLIB_MAJOR).$(SHLIB_MINOR)
SHARED_LIBS_LINK_EXTS=.so.$(SHLIB_MAJOR) .so
```

Now we can build and install:

```
make
make build-shared
make install
```

Install the shared libraries manually:

```
cp *.so* ~/arm/openssl/lib/
```

### libuuid

Get the source:

```
wget http://sourceforge.net/projects/libuuid/files/libuuid-1.0.3.tar.gz/download
mv download libuuid-1.0.3.tar.gz
tar xzf libuuid-1.0.3.tar.gz
cd libuuid-1.0.3/
```

Destination directory:

```
mkdir ~/arm/libuuid
```

Configure and make, using GNU Autotool's cross-compiling options:

```
./configure --build i686-pc-linux-gnu --host arm-linux-gnueabihf --prefix=/home/azureuser/arm/libuuid
make
make install
```

That wasn't so bad!

## Cross-compiling qpid-proton

Prepare the source for building:

```
git clone -b 0.8 https://github.com/apache/qpid-proton.git
cd qpid-proton
mkdir build
cd build
```

Copy the pre-requisites to the cross-compiling library directory:

```
sudo cp -r ~/arm/openssl/include/openssl /usr/arm-linux-gnueabihf/include
sudo cp -r ~/arm/openssl/lib/* /usr/arm-linux-gnueabihf/lib

sudo cp -r ~/arm/libuuid/include/uuid /usr/arm-linux-gnueabihf/include
sudo cp -r ~/arm/libuuid/lib/* /usr/arm-linux-gnueabihf/lib
```

Prepare a toolchain file for Cmake, e.g. cat the following to a file named `toolchain.arm`:

```
SET(CMAKE_SYSTEM_NAME Linux)
SET(CMAKE_SYSTEM_PROCESSOR arm)
#this one not so much
SET(CMAKE_SYSTEM_VERSION 1)
# specify the cross compiler
SET(CMAKE_C_COMPILER   arm-linux-gnueabihf-gcc)
SET(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)
# where is the target environment
SET(CMAKE_FIND_ROOT_PATH /usr/arm-linux-gnueabihf)
# search for programs in the build host directories
SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
# for libraries and headers in the target directories
SET(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

Run CMake from the `build` directory. We need to insist a little bit to convince CMake that we do have an OpenSSL library.

```
cmake .. -DCMAKE_TOOLCHAIN_FILE=~/toolchain.arm \
-DCMAKE_INSTALL_PREFIX=~/arm/qpid-proton \
-DSYSINSTALL_BINDINGS=OFF \
-DOPENSSL_ROOT_DIR=~/arm/openssl \
-DOPENSSL_INCLUDE_DIR=~/arm/openssl/include/openssl
```

Compile and install the library:

```
make
make install
```

Now you should have a properly compiled copy of Apache Qpid Proton compiled for ARM in your `~/arm/qpid-proton` directory.
