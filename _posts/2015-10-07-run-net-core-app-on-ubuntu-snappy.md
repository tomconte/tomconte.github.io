---
layout: post
date: 2015-10-07
title: Running a .NET Core application on snappy Ubuntu Core
---

Recently I have been playing with [snapcraft](https://developer.ubuntu.com/en/snappy/snapcraft/), Ubuntu's new tool designed to help developers package their applications for snappy Ubuntu Core. An intriguing idea was to explore how easy (or hard) it would be to use it to deploy .NET applications. The good news is that it turns out that it is fairly easy to use snapcraft to package and deploy a .NET application using our [.NET Core](https://github.com/dotnet/core) runtime environment for Linux!

.NET Core is basically a cross-platform implementation of .NET, which is originally driven by ASP.NET 5 workloads, but also by the goal of creating a modular, performant and cross-platform execution environment for any modern application. You can get started with .NET Core on Ubuntu very easily by following our [getting started guide](http://dotnet.github.io/core/getting-started/). This will guide through the steps of installing the .NET Version Manager (DNVM), the .NET Core Execution Environment (DNX) and Development Utilities (DNU), and finally writing and running a simple Hello World application.

Once you got this first app running, you will have to install snapcraft in order to package the application for Snappy. I like to keep track of the latest versions, since snapcraft is still very much under development, so I tend to install from the daily builds archive:

```
sudo add-apt-repository ppa:snappy-dev/tools-proposed
sudo apt-get update
sudo apt-get install snapcraft
```

Then to get you started I have published a [small sample on Github](https://github.com/tomconte/dnxsnap) that includes all the configuration required to package a .NET Core app:

```
git clone https://github.com/tomconte/dnxsnap.git
```

If you look into the `dnxhelloworld` directory, you will find the source code for the .NET app and a small Makefile used to build the app:

```makefile
all:
    dnu restore
    dnu publish --no-source --runtime active

install:
    cp -r bin/output ${DESTDIR}/dnxapp
    chmod -R +r ${DESTDIR}/dnxapp

clean:
    rm -rf bin
```

What this makefile does is use the `dnu restore` command to first restore the dependencies (NuGet packages), and then `dnu publish` to build the app and publish it as a self-contained package including everything that is necessary to run it, including the .NET runtime environment itself. The result is an application you can just copy over to another machine. The `install` target is for snapcraft, which will call it and expects the application to be copied into its destination directory.

The you will need a configuration file for snapcraft:

```yaml
name: dnx-sample
version: 1.0
vendor: Thomas Conte <tconte@microsoft.com>
summary: .NET Core example Snap
description: Runs .NET Core Hello World
icon: icon.png
binaries:
  hellodnx:
    exec: dnxapp/DNXHelloWorld

parts:
  dnxsample:
    plugin: make-project
    source: dnxhelloworld
    stage-packages:
      - libunwind8
```

This YAML file uses standard snapcraft constructs to build and package the app. The sample is described as a `make-project` which will cause snapcraft to call our Makefile targets. We also specify that we need to include the `libunwind8` package which is a depdency of the .NET Core runtime. The .NET Core publishing process creates an executable shell script for us, all we need to do is reference it in the `binaries` section so that snapcraft knows this is our entry point.

That's it! Now all you need to do is build the snap:

```
$ snapcraft
```

Copy the generated snap to a Snappy machine:

```
$ scp dnx-sample_1.0_amd64.snap ubuntu@mysnappyvm.cloudapp.net:
```

On the Snappy machine, install the snap and run it:

```
$ sudo snappy install dnx-sample_1.0_amd64.snap --allow-unauthenticated
$ dnx-sample.hellodnx
```

Hope you will find some interesting use cases for .NET Core on snappy Ubuntu Core, please [let me know](https://twitter.com/tomconte) if you build anything interesting!
