---
layout: post
date: 2014-09-15
title: Using the Alljoyn Thin SDK with Visual Studio 2013 and the Intel Galileo
---

As part of our exploration of the Alljoyn framework, we have spent some time studying the Alljoyn Thin Client and how it can be used on the Microsoft platform. The Alljoyn Thin Client (AJTC) is a lightweight, stand-alone implementation of the Alljoyn protocol, suitable for using in limited or contrained environments like embedded devices. Our end goal was to get AJTC running on the Intel Galileo board running Windows for Devices, so that we could build an end-to-end scenario showing an Alljoyn standard client, running on Windows 8, communicating with a Galileo device running the thin client.

## Compiling with Visual Studio 2013 and SCons

We started from the publicly available bits on the [Allseen Alliance download center](https://allseenalliance.org/source-code). Since we wanted to rebuild AJTC for a new target, we started with the full source code archive: [alljoyn-suite-14.06.00-src.tar.gz](https://allseenalliance.org/releases/alljoyn/14.06.00/alljoyn-suite-14.06.00-src.tar.gz). This archive contains the source code both for the standard client (in `core/alljoyn`) and the thin client (in `core/ajtcl`).

The very minimum you will need to rebuild AJTC is the SCons build tool and its prerequisite, Python 2.7. I used the latest version of SCons, 2.3.3, and I suggest you do the same, as it seems to fix some problems when building with Visual Studio 2013. If you install Python 2.7 using the standard Windows installer, then the SCons installer for Windows will automatically install itself in the right place, i.e. `C:\python2`. All you need to check is that `C:\python2\Scripts` is in your `PATH`, since this is where the SCons executable will be installed.

Currently (in version 14.06), the AJTC does not build out of the box with Visual Studio 2013 (it requires VS 2012). However that is really easy to change, since the build tool does have support for VS 2013. All you need to do is to edit the Thin Client's SConscript in `core/ajtcl` to add '12.0' to the list of `allowed_values` in `MSVC_VERSION`.

```
vars.Add(EnumVariable('MSVC_VERSION', 'MSVC compiler version - Windows', default_msvc_version, allowed_values=('8.0', '9.0', '10.0', '11.0', '11.0Exp', '12.0')))
```

You can then run the build using the following command:

```
scons WS=off MSVC_VERSION=12.0
```

This should allow you to compile the library and samples. In order to test the samples though, you will also need to run an Alljoyn daemon on your machine, since the thin clients do not include a bundled daemon. You can find a sample daemon in the full Alljoyn SDK: download [alljoyn-14.06.00-win7x64vs2012-sdk.zip](https://allseenalliance.org/releases/alljoyn/14.06.00/alljoyn-14.06.00-win7x64vs2012-sdk.zip) (or the equivalent 32-bit archive), unzip it, and you will find an executable named `SampleDaemon.exe` in `cpp\bin\samples`. You can run this daemon with the `-n` argument to pass the name of the bus you want to create, e.g.:

```
SampleDaemon.exe -n org.alljoyn.Bus.sample
```

Then you can run the thin client `basic_service.exe` to expose the service on the bus, and `basic_client.exe` to call it.

## Porting the Thin Client library to the Intel Galileo

Now the next step we took was to port and compile the AJTC for the Intel Galileo. The Galileo is an x86-based prototyping board, that you can [update](https://ms-iot.github.io/content/IBoughtAGalileo.htm) to run a specific version of Windows called "Microsoft Windows for the Windows Developer Program for IoT". You can then program from Visual Studio using the [dedicated tools](https://ms-iot.github.io/content/SetupPC.htm) from the Developer Program for IoT. In this release, you can program in C or C++, and you will find a number of [sample applications](https://ms-iot.github.io/content/SampleApps.htm) on the site.

When you install the tools, you will see a new a new template in Visual C++ &rarr; Windows for IoT &rarr; Galileo Wiring App. This template initializes the C++ project in Visual Studio so that you have all the right parameters required to compile for the Intel Galileo as a target. It will also give you a couple C++ and header files implementing a Wiring example, Wiring being the C++ API used by the Arduino development tools.

In our case, we can use this template as a base to port the AJTC code across. The steps are as follows:

- Create a new project using the Galileo Wiring App template
- In the project properties, set the project type to "Static library"
- Remove the existing sample C++ items in "Source Files" and "Header Files"
- Add the *.c source code files in `core\ajtcl\src` to the project, by right-clicking "Source Code" add selecting "Add &rarr; Existing Item" and selecting all the .c files
- Also add the following source files:

 - *.c in `core\ajtcl\target\win32`
 - *.c in `core\ajtcl\crypto\ecc`
 - *.c in `core\ajtcl\external\sha2`
 - *.c in `core\ajtcl\crypto`

- Add the following header file directories in "C/C++ &rarr; All Options &rarr; Additional Include Directories" in the project properties:

 - `core\ajtcl\inc`
 - `core\ajtcl\target\win32`
 - `core\ajtcl\external\sha2`

- Add `_CRT_SECURE_NO_WARNINGS` to "Preprocessor Definitions" in the project properties

You should now be able to compile the project. This will generate the lib file you can use in your projects e.g. `GalileoAJTC\Debug\GalileoAJTC.lib`.

## Compiling and running a basic sample

Now that we have compiled the AJTC libary for the Galileo, let's try to build an Alljoyn Service to show that we can communicate with the Bus and the Client running on the Windows machine. We can for example port the `basic_service.exe` sample that we used previously.

For that, let's create another another Galileo Wiring App in Visual Studio. We are going to copy/paste and adapt the code from the sample source code in `core\ajtcl\samples\basic\basic_service.c`.

- Copy/paste the basic service code in Main.cpp
- Add the include files as above
- Add `_CRT_SECURE_NO_WARNINGS` as above
- Add the library `GalileoAJTC.lib` to "Linker &rarr; All Options &rarr; Additional Dependencies"
- Add the library directory where your .lib file was compiled to "VC++ Directories &rarr; Library Directories"

Now you must surround the Alljoyn include directives with `extern "C"`:

```
extern "C" {
#include <aj_debug.h>
#include "alljoyn.h"
}
```

And you will need to modify the `main` function to match the Arduino/Wiring conventions:

```
int _tmain(int argc, _TCHAR* argv[])
{
	return RunArduinoSketch();
}

void setup()
{
	// initialize the digital pin as an output.
	pinMode(13, OUTPUT);

	Log(L"Starting up");

	digitalWrite(13, LOW);
}

// the loop routine runs over and over again forever:
void loop()
{
	AJ_Main();
}
```

You should now be able to build the sample. To run in on the Galileo, your board must be powered up and connected to the network. It should show up in the Galileo Watcher application with a number of details, like the board name and the IP address. Before you try to run the sample, make sure that you type the appropriate board name or IP address in the "Remote Server Name" in the "Debugging" project properties section. This is the address where Visual Studio will try to connect to execute the program.

When you click on "Remote Windows Debugger", you should be prompted for your Galileo credentials, and your program will be executed on the board.

To test, make sure you have first started the Sample Daemon as instructed above. You can then launch the `basic_client.exe` sample on the PC and it should be able to communicate with the service running on the board!

Another way to start the service is via telnet, directly to the board. From your Galileo Watcher window, right-click on your board and select the "Telnet Here" option (you must first install the telnet client via "Programs and Features"). You will be prompted for your credentials, and you will be logged into your board! You will find your executables in `C:\test` and you can run them manually:

```
C:\test
GalileoAJTCSample.exe
```

This will allow you to see the standard output from the application.

You could now modify the service and client samples to do something slightly more useful, like turning a LED on or off! This will be the subject of further blog posts.
