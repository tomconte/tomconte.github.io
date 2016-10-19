---
title: Building a Windows development box for Ethereum and Truffle
layout: post
---

In the last few weeks I have been working on a number of Ethereum smart contract projects, where I have come to rely on the [Truffle](https://truffle.readthedocs.io/en/latest/) development tool to structure my application, as well as compile, test, and deploy the smart contracts.

However I have realized that installing Truffle on a Windows machine implied quite a number of pre-requisites if you start from a base machine with no development tools installed... The short list is:

- [Node.JS](https://nodejs.org/) & the very latest version of npm
- Git
- A working [node-gyp](https://github.com/nodejs/node-gyp), which on Windows implies a bunch of other dependencies like the C++ Build Tools and Python 2.7
- OpenSSL, specifically an older 1.0.2 version which includes `libeay32.lib`

So I developed a [PowerShell script](https://github.com/tomconte/TruffleDevBox/blob/master/Install-PreRequisites.ps1) that automates the installation of all these dependencies. Most of the work is actually performed by Felix Rieseberger's amazing [windows-build-tools](https://github.com/felixrieseberg/windows-build-tools) package.

I also have a [Vagrantfile](https://github.com/tomconte/TruffleDevBox/blob/master/Vagrantfile) to start up a development VM locally (on both Hyper-V and VirtualBox).

Once the VM is up and running, Truffle should be installed and ready to go. You should be able to add other blockchain development tools like TestRPC with no problems.

You will also need to install your favorite code editor, I would suggest [Visual Studio Code](https://code.visualstudio.com/) and its [Solidity extension](https://github.com/juanfranblanco/vscode-solidity)!

My TODO list includes adding some Azure options, like an ARM template and/or DevTest Labs artifacts.

Hope this helps!

