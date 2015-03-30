---
layout: post
date: 2015-03-30
title: Let's cross-compile Qpid Proton for the Arduino Yun
---

Exploring further the dark art of cross-compilation, today I would like to present you some simple instructions on how to cross-compile the [Apache Qpid Proton](https://github.com/apache/qpid-proton) library to run on the Arduino Yun integrated [OpenWRT-based gateway](http://arduino.cc/en/Guide/ArduinoYun#toc2). The idea here is that you can then execute a process on the Linux side of the Yun and communicate from your micro-controller code via the Bridge library to send AMQP commands to Azure Event Hubs.

Fortunately, [MS Open Tech China has published on its Github account](https://github.com/msopentechcn/qpid-proton-c-openwrt) excellent instructions on how to do this. However, they are building the OpenWRT SDK from scratch whereas I would like to present you simplified instructions using the pre-built [OpenWRT SDK](http://wiki.openwrt.org/doc/howto/obtain.firmware.sdk).

From a brand new Ubuntu 14.10 machine, first install a bunch of dependencies and development tools:

```
sudo apt-get update

sudo apt-get install -y git subversion build-essential asciidoc \
	fastjar flex gawk libgtk2.0-dev intltool zlib1g-dev \
	genisoimage libncurses5-dev libssl-dev ruby sdcc unzip \
	bison libboost-dev libxml-parser-perl libusb-dev bin86 \
	bcc sharutils openjdk-7-jdk mercurial cvs bzr \
	nodejs-legacy npm curl
```

(this will take a long time, but it gives you everything you need to build stuff and more!)

Download the latest pre-compiled OpenWRT SDK from this URL:

http://downloads.openwrt.org/snapshots/trunk/ar71xx/generic/

e.g. in my case:

```
wget http://downloads.openwrt.org/snapshots/trunk/ar71xx/generic/OpenWrt-SDK-ar71xx-generic_gcc-4.8-linaro_uClibc-0.9.33.2.Linux-x86_64.tar.bz2
```

Prepare the Qpid-Proton package following the [MSOpenTechCN instructions](https://github.com/msopentechcn/qpid-proton-c-openwrt)

```
cd package/
mkdir openwrt-qpid-proton
cd openwrt-qpid-proton
git clone https://github.com/apache/qpid-proton.git -b 0.8
mv qpid-proton src
vi Makefile
vi src/proton-c/CMakeLists.txt
cd ../..
```

Careful: when you copy-paste the OpenWRT Makefile example from the instructions, **you need to transform all leading spaces into tabs**, otherwise make won't understand the file. In vi for example, use the following command. There are also a few double-spaces you will need to turn into tabs as well.

```
:%s/        /^I/g
```

Add & build libopenssl and libuuid dependencies:

```
./scripts/feeds update -a
./scripts/feeds install libopenssl libuuid
make package/feeds/base/openssl/compile
make package/feeds/base/util-linux/compile
```

Copy the necessary OpenSSL and libuuid headers and libraries into the appropriate toolchain directories:

```
export TC=~/OpenWrt-SDK-ar71xx-generic_gcc-4.8-linaro_uClibc-0.9.33.2.Linux-x86_64/staging_dir/toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2

mkdir $TC/include/uuid
cp build_dir/target-mips_34kc_uClibc-0.9.33.2/util-linux-2.25.2/libuuid/src/uuid.h $TC/include/uuid

cp build_dir/target-mips_34kc_uClibc-0.9.33.2/util-linux-2.25.2/.libs/libuuid.* $TC/lib

cp -r -L build_dir/target-mips_34kc_uClibc-0.9.33.2/openssl-1.0.2a/include/openssl $TC/include

cp build_dir/target-mips_34kc_uClibc-0.9.33.2/openssl-1.0.2a/libcrypto.so* $TC/lib
cp build_dir/target-mips_34kc_uClibc-0.9.33.2/openssl-1.0.2a/libssl.so* $TC/lib
```

Unfortunately, there is currently [a bug with CMake](http://www.cmake.org/gitweb?p=cmake.git;a=commitdiff;h=c5d9a8283cfac15b4a5a07f18d5eb10c1f388505) where it crashed when trying to find the newer version of OpenSSL, 1.0.2. So, if CMake fails with errors like "list GET given empty list" during OpenSSL detection, you will have to patch the `FindOpenSSL.cmake` file by hand, following the diff referenced via the link above.

```
vi staging_dir/host/share/cmake-2.8/Modules/FindOpenSSL.cmake
```

You can now compile the Proton package using the OpenWRT-specific Makefile:

```
make V=s package/openwrt-qpid-proton/compile
```

Resulting package:

```
bin/ar71xx/packages/base/openwrt-qpid-proton_0.8-1_ar71xx.ipk
```

Next up, how to use this library from the Yun!
