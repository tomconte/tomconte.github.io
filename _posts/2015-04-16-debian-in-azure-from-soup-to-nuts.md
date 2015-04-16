---
layout: post
date: 2015-04-16
title: Running Debian in Azure, from soup to nuts
---

Here are step-by-step instructions on creating your own Debian image to run in Microsoft Azure. This is not that complicated and will allow you to fully customize your OS and select only the packages you need, following the great Debian minimalist tradition!

I am required to point out that the resulting VM Image, while fully functional in Azure, falls under the following important notice:

> The Azure platform SLA applies to virtual machines running the Linux OS only when one of the [endorsed distributions](http://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-endorsed-distributions/) is used. All Linux distributions that are provided in the Azure image gallery are endorsed distributions with the required configuration.

# Install Debian in Hyper-V

First, you need to install Debian locally in Hyper-V. We want a minimal install, so just go to [www.debian.org](https://www.debian.org//) and download the latest network installer, in my case [debian-7.8.0-amd64-i386-netinst.iso](http://cdimage.debian.org/debian-cd/7.8.0/multi-arch/iso-cd/debian-7.8.0-amd64-i386-netinst.iso).

Then create the VM in Hyper-V:

- New --> Virtual Machine, Next
- Give it a name, Next
- Select "Generation 1", Next
- Keep the startup memory at 1024MB, Next
- Do not select a connection, we will do that later. Next.
- Select "Attach a virtual disk later" since we want to customize this. Next.
- Finish.

Now select your new VM and click on Settings. We need to change a few things there. Like removing the default Network Adapter:

- Select the Network Adapter that was created originally and click "Remove", then "Apply".

And adding a Legacy one:

- Select "Add Hardware", "Legacy Network Adapter", "Add".
- Connect it to a Virtual Switch so that it can access the Internet. Click "Apply".

Then adding a disk:

- Select "IDE Controller 0", "Hard Drive", "Add".
- In "Virtual hard disk", click "New".
- Next.
- Select "VHD" (Azure does not support VHDX). Next.
- Select "Fixed size". Next.
- Give a name to your VHD file. Next.
- Select "Create a new blank virtual hard disk", and enter a size of 30GB. You can select a smaller size if you want to save on uploading time!
- Finish. OK.

Your VM is now ready! Let's install Debian.

- Click on "Connect..." to open your VM's console.
- Media --> DVD Drive --> Insert Disk...
- Select the Debian ISO file.
- Click on "Start".

We are now in the Debian installation process, let's run through the steps to install a very minimal system!

- Select "64 bit install"
- In the language selections screens, you can pick your favorite settings, I will go for:
 - English
 - United States
 - American English
- Choose a hostname, e.g. "debian". It will be modified by Azure when you will create a VM, so it does not really matter.
- Choose a domain name, e.g. "foo.com". Here again this will be modified by Azure.
- Choose a root password. It will be deleted by the Azure provisioning process, but you will need it to prepare the image!
- Choose a full name for a user account. Azure will create a user account for you, so you can choose whether to keep this one or not. I will use "Debian User".
- Choose a usernam, e.g. "debian".
- Choose a password for your user.
- Select a time zone, I will pick Pacific.

We are now in the disk configuration steps.

- For the partitioning method, choose "Manual" since we want to customize this.
- Select the virtual hard disk device, it should be called something like "Msft Virtual Disk".
- Confirm that you want to create a new partition table ("Yes").
- Select the "FREE SPACE" line.
- Select "Create a new partition".
- Keep the default partition size (the full disk).
- Type: "Primary".
- Select "Done setting up the partition".
- Select "Finish partitioning and write changes to disk".
- The installer warns you that you have not provisioned any swap space. This is fine, Azure can do that for you. Select "No" (don't return to the partitioning menu).
- "Write the changes to disk"? Say "Yes"!

The installer will start installing the base system.

Select a country close to you to download additional packages, an archive mirror, and HTTP proxy information (if required).

When asked about "popularity contest", I chose "No".

In the software selection screen, unselect (e.g. un-star) everything. We will install what we need manually in order to keep the installation minimal.

Select "Yes" to install the GRUB boot loader.

Select "Continue" to boot in your new Debian VM!

# Prepare Debian for Azure

Now that we have a minimal Debian VM running, we need to add a few things in order to be able to run in Azure: the `sshd` demon, `sudo`, and the [Linux Azure Agent](http://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-agent-user-guide/).

Log in as `root` in your VM.

```
apt-get install -y sudo openssh-server
```

The Linux Agent has a few dependencies we need to install:

```
apt-get install -y python parted
```

We also need CA certificates in order to be able to use WGet with SSL:

```
apt-get install -y ca-certificates
```

Then the Agent itself, I recommend checking the [releases in the Github repo](https://github.com/Azure/WALinuxAgent/releases) and grab the latest one.

```
wget https://github.com/Azure/WALinuxAgent/archive/WALinuxAgent-2.0.12.tar.gz
tar xzf WALinuxAgent-2.0.12.tar.gz
mv WALinuxAgent-WALinuxAgent-2.0.12/waagent /usr/sbin
rm -rf WALinuxAgent-*
```

Then initialize the Agent:

```
chmod 755 /usr/sbin/waagent
/usr/sbin/waagent -install -verbose
```

Then we recommend a few kernel boot parameters to facilitate logging the console messages to the Azure infrastructure.

Edit the `/etc/default/grub` file and modify the `GRUB_CMDLINE_LINUX` parameter as follows:

```
GRUB_CMDLINE_LINUX="console=ttyS0 earlyprintk=ttyS0 rootdelay=300"
```

Then run:

```
update-grub
```

Now everything is ready for Azure. You can reboot the machine and check everything starts automatically, e.g. `sshd` and the Agent.

The final steps before uploading the VHD to Azure: you will prepare the VM, using the Agent. This preparation step is described in detail in the [Agent README](https://github.com/Azure/WALinuxAgent) file; it deletes the following:

- All SSH host keys (if Provisioning.RegenerateSshHostKeyPair is 'y' in the configuration file)
- Nameserver configuration in /etc/resolv.conf
- Root password from /etc/shadow (if Provisioning.DeleteRootPassword is 'y' in the configuration file)
- Cached DHCP client leases.
- Resets host name to localhost.localdomain.

Before you do this, you should also check the README for other interesting configuration settings you can customize in `/etc/waagent.conf`, for example `ResourceDisk.EnableSwap` if you want to add some swap space at provisioning time. 

As a precaution, I would also advise authorizing your local user to `sudo` in case you get locked out of the root account (remember the Agent will remove the root password!). Edit `/etc/group` and add your username to the `sudo` group.

```
waagent -force -deprovision
export HISTSIZE=0
halt
```

# Upload the Debian VHD to Azure

In order to upload your VHD to Azure, you will need to use the Azure PowerShell cmdlets. Make sure your PowerShell environment is configured to connect to your Azure subscription. The command is the following:

```
Add-AzureVhd -Destination https://tconeu.blob.core.windows.net/vhdstore/debian-azure.vhd -LocalFilePath "C:\User
s\Public\Documents\Hyper-V\Virtual hard disks\DebianAzure.vhd"
```

Just change the Destination parameter to point to an existing Storage Account, and the LocalFilePath to the VHD you created in Hyper-V. The upload process can take a while, so it's probably the right time to go out and grab something to drink :)

# Create the VM Image

Since we are already in PowerShell, let's use that to create the VM Image based on our VHD. You can also do that using the Web portal if that is your cup of tea!

```
Add-AzureVMImage -ImageName Debian -MediaLocation https://tconeu.blob.core.windows.net/vhdstore/debian-azure.vhd -OS Linux
```

# Create a Debian VM

You can now create a Debian VM using the method of your choice (portal, PowerShell, cross-platform CLI, REST API, remote mind control, etc.)

Make sure you can log in, `sudo`, etc. Don't forget to remove the local user you created above, if you do not want to keep it (remember, it has `sudo` privileges!). Depending on which region you create the VM, you might also want to adjust the URLs in `/etc/apt/sources.list`.
