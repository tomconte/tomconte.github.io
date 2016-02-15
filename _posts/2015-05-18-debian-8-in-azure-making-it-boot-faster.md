---
layout: post
date: 2015-05-18
title: Running Debian 8 "Jessie" in Azure - making it boot faster
---

I have recently uploaded a hand-made version of [Debian "8" Jessie for Azure in the VMDepot repository](https://vmdepot.msopentech.com/Vhd/Show?vhdId=52539). It was before I discovered [bootrap-vz](https://github.com/andsens/bootstrap-vz) and its ability to automatically generate Debian images for various virtualised and Cloud environments!

While creating the image locally in Hyper-V, I ran into a problem where the image would always take 5 minutes to boot. This was obviously linked to the following line that we [recommend](http://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-create-upload-vhd-generic/) adding to the boot parameters:

~~~
console=ttyS0 earlyprintk=ttyS0 rootdelay=300
~~~

The culprit is obviously **rootdelay=300**: when this parameter is present Debian will always wait for five minutes for the boot disk to be ready. However that was not really the intent of the parameter: the idea was to have a **maximum** timeout of 5 minutes for the boot disk to be ready, just in case something runs a bit slow in the VM provisioning process.

In Debian 8, this delay is implemented in `/usr/share/initramfs-tools/scripts/init-top/udev`:

~~~
# If the rootdelay parameter has been set, we wait a bit for devices
# like usb/firewire disks to settle.
if [ "$ROOTDELAY" ]; then
	sleep $ROOTDELAY
fi
~~~

As you can see, this will simply sleep for five minutes, which causes our delay.

In the Ubuntu images that we have in the Azure Gallery, this has been fixed by removing this delay altogether (as part of a larger reorg of the init scripts), as described in this [Ubuntu systemd bug report](https://bugs.launchpad.net/ubuntu/+source/systemd/+bug/1202700). 

As you can see in Debian's `/usr/share/initramfs-tools/scripts/local`, the `rootdelay` parameter is still used as a maximum time to wait for the boot device to be ready:

~~~
# If the root device hasn't shown up yet, give it a little while
# to allow for asynchronous device discovery (e.g. USB).  We
# also need to keep invoking the local-block scripts in case
# there are devices stacked on top of those.
if [ ! -e "$1" ] || ! $(get_fstype "$1" >/dev/null); then
	log_begin_msg "Waiting for $2 file system"
	
	# Timeout is max(30, rootdelay) seconds (approximately)
	slumber=30
	if [ ${ROOTDELAY:-0} -gt $slumber ]; then
		slumber=$ROOTDELAY
	fi
~~~

So I felt it was safe to just comment out the offending `sleep` command in the init scripts of my Debian 8 VM image.
