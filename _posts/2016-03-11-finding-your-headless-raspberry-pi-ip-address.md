---
layout: post
date: 2016-03-11
title: Finding your headless Raspberry Pi IP address at boot time
---

When working on various Raspberry Pi projects, I mostly use the little buggers in "headless" mode, e.g. with no screen, keyboard or mouse connected. This means I have no obvious way of finding the device's IP address at first boot after a clean re-installation of the OS, other than using brute-force methods like [scanning all my network in anger](http://angryip.org/), looking for some Raspberry Pi MAC addresses or open SSH ports. The scanning approach might work on my home network, but is not a very good idea when I am working back at the mothership, where we have thousands of devices connected to the LAN.

Here are my notes on how to solve this problem, using a Raspberry Pi running [Raspbian Jessie](https://www.raspberrypi.org/downloads/raspbian/).

# First boot

The first trick is logging on the Pi after its first boot. For that, I use a *direct Ethernet connection between my Windows laptop and the Raspberry Pi*: with the Pi turned off, just use a regular Ethernet cable and connect the two Ethernet ports together. Thanks to [Auto MDI-X](https://en.wikipedia.org/wiki/Medium-dependent_interface#Auto_MDI-X), you don't have to worry about using a crossover or patch cable. Then, power on the Pi.

Thanks to the [APIPA (Automatic Private Internet Protocol Addressing)](https://wiki.wireshark.org/APIPA) process, your Windows PC will receive a random IP address in the 169.254.0.0/16 range; for example, I got 169.254.197.190. We are going to use this "link local" range to communicate with our Pi: what we need is a way to assign it an IP in that range. By default the Pi is configured to use DHCP, so we are going to install and run a small DHCP server to get it configured!

I have been using [DHCP Server for Windows](http://www.dhcpserver.de/cms/), which is a small, simple and portable DHCP server. Just download and extract the ZIP file, then run the `dhcpwiz.exe` configuration wizard.

- In the "Network interface cards" dialog, *make sure to select the network card with a 169.254 IP address*: you don't want to be messing up your home or work network with a rogue DHCP server! 
- You don't need to select anything in the "Supported Protocols" dialog
- In the "Configuring DHCP for Interface" dialog, in the "IP-Pool" configuration, make sure you have `169.254.0.1 - 255` 
- In the next dialog, click on "Write INI file" 
- In the last dialog, click on "Run DHCP server immediately" and click "Finish"

In the DHCP Server launch dialog, click on "Continue as tray app" to run the server right away.

Wait a few seconds, and our little DHCP server should display notifications saying that it just allocated some IP addresses to our two machines: the Windows PC will get a new IP address (probably 169.254.0.1) and the Pi will get one too! (probably 169.254.0.2)  

If nothing happens, try kicking off the DHCP process by unplugging the Ethernet cable from the Pi and plugging it back in (while keeping the Pi turned on).

Sometimes, depending on the timing, the two IP addresses might also be allocated the other way around, e.g. 169.254.0.2 to the PC and 169.254.0.1 to the Pi. If you have any doubt, check the `dhcptrc.txt` log file which logs all the DHCP activities.

Now your Pi should be accessible via the "link local" address! Try it out:

~~~
ssh pi@169.254.0.2
~~~

# Getting notified of your final IP address

Now that you have access to your Pi, you want to put something in place so that you can figure out its final IP address when you plug it into a real Ethernet LAN. The trick here is that the Raspbian Jessie network management system is a bit different from a stock Debian distro, e.g. the usual ` /etc/network/if*` scripts are not used.

Instead, Raspbian uses [`dhcpcd`](http://roy.marples.name/projects/dhcpcd/index) to handle network configuration in a simplified manner, which causes much confusion if you are used to the usual Debian way of things.

It turns out that `dhcpcd` has a very neat way of hooking up into the network configuration: all you need to do is drop a script in the `/lib/dhcpcd/dhcpcd-hooks` directory, and it will get called at the various stages of the IP allocation process. The script will receive a bunch of useful variables like `${interface}` and `${new_ip_address}` that make it easy to get all the data we need. Check the `dhcpcd-run-hooks` man page if you want all the gory details!

So, in our case, all we want to do is get notified when an IP address is allocated to our of our network interfaces. "But how?!" I hear you ask. Of course we could use a simple e-mail, but why not use something more modern like getting a notification on our phone, right?

In my case, I opted for the [Pushover](https://pushover.net/) service which I have been using for other projects, but you could use any other push notification service like [Pushalot](https://pushalot.com/), [Pushbullet](https://www.pushbullet.com/), etc. Pushover is easy to set up: after you sign up you will receive a User Token, and you will then need to create an application to get an API Token. Of course, you will also need to install the app on your smartphone!

Pushover has a really simple [REST API](https://pushover.net/api) that you can call using `curl`, but in order to make my life even simpler I used a nice little wrapper script called [pushover.sh](https://github.com/jnwatts/pushover.sh). Just download the script somewhere, e.g. in the `root` home directory:

~~~ sh
mkdir scripts
cd scripts
wget https://raw.githubusercontent.com/jnwatts/pushover.sh/master/pushover.sh
chmod +x pushover.sh
~~~

Then drop a script like this one in `/lib/dhcpcd/dhcpcd-hooks`; I named mine `99-pushover` so it is called last.

~~~ sh
if $if_up; then
  /root/scripts/pushover.sh -U user_key -T api_token -t "Raspberry Pi ${interface} up" "IP address for ${interface} is ${new_ip_address}"
fi
~~~

Now all you need to do is unplug your Pi from the direct-to-PC Ethernet connection, and plug it into a real (i.e. Internet-connected) LAN; the DHCP process should trigger automatically, no need to reboot. And the Pi gets an IP address via DHCP, you should get a notification!
