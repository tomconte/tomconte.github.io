---
layout: post
date: 2014-03-20
title: Deploying Cloudera on Windows Azure Virtual Machines using Cloudera Manager
---

Introduction
------------

In this tutorial, I would like to show you how to install CDH, the [Cloudera Hadoop distribution](http://www.cloudera.com/content/cloudera/en/products/cdh.html) on Windows Azure Virtual Machines, using their automated installer and management toold called [Cloudera Manager](http://www.cloudera.com/content/cloudera/en/products/cloudera-manager.html).

For this tutorial, since we will be deploying the platform on Linux VMs, I am going to assume that we are going to install everything from a Linux command line exclusively, using our [cross-platform CLI tools](https://github.com/WindowsAzure/azure-sdk-tools-xplat) and the Management Portal Web interface when required. In other words: you don't need a Windows machine to follow this tutorial ;-)

In summary, here are the steps that we are going to follow:

- Preparing our SSH keys for authentication on the cluster
- Creating the Virtual Network that defines which internal IP addresses the VMs will use
- Creating our VMs
- Configuring the DNS so that all machines can resolve their internal names and IP addresses
- Run the Cloudera Manager installer to create the Hadoop cluster

I hope you enjoy this document, and please send feedback!

Preparing our SSH keys
----------------------

First, let's create an SSH key and certificate to use for authentication. This will be useful in order to log on to all our VMs without being prompted for passwords. You can find a more detailed SSH procedure in the [Windows Azure documentation for Linux Virtual Machines](http://www.windowsazure.com/en-us/manage/linux/how-to-guides/ssh-into-linux/). This is slightly different from what you would do usually to generate new SSH keys (i.e., use `ssh-keygen`).

~~~
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout myPrivateKey.key -out myCert.pem
Generating a 2048 bit RSA private key
....+++
.......................................................+++
writing new private key to 'myPrivateKey.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:FR
State or Province Name (full name) [Some-State]:92
Locality Name (eg, city) []:Issy-les-Moulineaux
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Microsoft
Organizational Unit Name (eg, section) []:DPE
Common Name (e.g. server FQDN or YOUR name) []:Thomas Conte
Email Address []:tconte@microsoft.com
~~~

Windows Azure needs the SSH public key to be embedded in a PEM formatted certificate, but you could enter any information you want, it doesn't matter. It's only the generated public key we are interested in!

Let's secure the key and move it to our `.ssh` directory so that it used by default.

~~~
$ chmod 600 myPrivateKey.key
$ mv myPrivateKey.key ~/.ssh/id_rsa
~~~

I have found that Cloudera Manager does not like the private key format very much though. Let's convert it right now and keep the generated file somewhere, we will need it later:

~~~
$ openssl rsa -in ~/.ssh/id_rsa -out PrivateKeyCDH
~~~

Creating the Virtual Network
----------------------------

We will use the cross-platform command-line tools to create our VMs. If you haven't got them installed, first go through the steps to install the [Windows Azure Command-Line Tools for Mac and Linux](http://www.windowsazure.com/en-us/develop/nodejs/how-to-guides/command-line-tools/). I won't go into the details here, but once installed you should be able to type `azure` from the command line and get a friendly response back, plus you must import your publish settings so that the CLI is connected to your Windows Azure subscription.

**Update**: you can now create a Virtual Network using the command-line tool.

Create the Affinity Group:

~~~
azure account affinity-group create --location "West Europe" ezwest
~~~

Register the DNS server:

~~~
azure network dnsserver register -i mycdh 10.0.0.4
~~~

Create the Virtual Network:

~~~
azure network vnet create --address-space 10.0.0.0 --cidr 8 --subnet-name cdh --subnet-start-ip 10.0.0.0 --subnet-cidr 24 --affinity-group ezwest --dns-server-id mycdh cdhvnet
~~~

Read on if you want to understand more about the Virtual Network configuration, otherwise you can just skip to the next step!

The first thing we need to create is an Affinity Group, a kind of alias that will represent the datacenter where all our infrastructure will be created. In the Portal, on the left-hand side, at the bottom of the menu, click on Settings, then click on the Affinity Groups tab at the top of the main area. Click on Add an Affinity group, give it a name (I will use "cdh-AG") and select the datacenter (I will use North Europe).

The simplest way to create a complete Virtual Network configuration is to import a ready-made configuration file. Here is the one I used, you can save it into a file and edit it as needed. It is also possible to create a Virtual Network "manually" from the Portal and then export this configuration to a file to inspect or modify it (see Annex 1).

If you look at the XML configuration file below, you will find the following information:

- The Virtual Network name, `cdh`, and the Affinity Group where it will be created, `cdh-AG`
- The address space that will be used to assign IP addresses to the Virtual Machines (`10.0.0.0/20`)
- Within this address space, one or several subnets the VMs will be deployed in, `10.0.0.0/23`, each being given a name that you can use when configuring the Virtual Machines (`Subnet-1`)
- The IP address of a DNS server (more about this below), which of course resides within the address space of the Virtual Network, i.e. `10.0.0.4` in this case
- A reference from the Virtual Network to the DNS server: all VMs will get assigned this DNS when they request their network configuration from Windows Azure

The trick here is: how did I choose `10.0.0.4` as the address of the DNS server? I know by experience that Windows Azure will allocate IP addresses sequentially within the Subnet, and that in this case, the first one it will allocate is `10.0.0.4`. In short, this means that the first VM I start will become the DNS for the rest of the machines in the Virtual Network.

This is really a shortcut, and the longer way is to create the Virtual Network in the Portal, start the first VM, make a note of its IP address, and then register it as a DNS using _New_, _Networks_, _Virtual Network_, _Register DNS Server_. Importing the configuration file is just a faster way to get everything configured!

And now the configuration file:

~~~xml
<NetworkConfiguration xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.microsoft.com/ServiceHosting/2011/07/NetworkConfiguration">
  <VirtualNetworkConfiguration>
    <Dns>
      <DnsServers>
        <DnsServer name="mycdh" IPAddress="10.0.0.4" />
      </DnsServers>
    </Dns>
    <VirtualNetworkSites>
      <VirtualNetworkSite name="cdh" AffinityGroup="cdh-AG">
        <AddressSpace>
          <AddressPrefix>10.0.0.0/20</AddressPrefix>
        </AddressSpace>
        <DnsServersRef>
          <DnsServerRef name="mycdh" />
        </DnsServersRef>
        <Subnets>
          <Subnet name="Subnet-1">
            <AddressPrefix>10.0.0.0/23</AddressPrefix>
          </Subnet>
        </Subnets>
      </VirtualNetworkSite>
    </VirtualNetworkSites>
  </VirtualNetworkConfiguration>
</NetworkConfiguration>
~~~

Make sure you use your Affinity Group name in the AffinityGroup attribute of the VirtualNetworkSite element!

To import this configuration, in the Portal, go to _New_, _Networks_, _Virtual Network_, and select _Import Configuration_. The Virtual Network will be created and will then appear in the Networks module in the Portal.

Creating our Admin/DNS VM
-------------------------

We will prepare a Storage Account in our new Affinity Group, where the Virtual Machine disks will be stored. We can do this from the command line:

~~~
$ azure account storage create --affinity-group cdh-AG cdhstorage
~~~

Now let's find a good Linux image to use. You can use the following command to list all current images.

~~~
$ azure vm image list
~~~

In my case I am going to choose Ubuntu 12.04:

~~~
b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-12_04_2-LTS-amd64-server-20130415-en-us-30GB
~~~

Now let's create our first VM. This VM will become our DNS server and will also be used to run the Cloudera Manager. For this first step we ask for a "small" size since we don't need to run much stuff. We specify that we want to open the SSH port for remote shell, and add our SSH certificate so that the VM is properly setup for authentication. We also specify our Virtual Network and Subnet so that the VM gets assigned an IP address from the prefix we selected. Finally, we need to specify the Affinity Group, obviously the same we used when creating the Virtual Network.

~~~
$ azure vm create --virtual-network-name cdh --subnet-names Subnet-1 --affinity-group cdh-AG --vm-size small \
--ssh 22 --ssh-cert ./myCert.pem mycdh \
b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-12_04_2-LTS-amd64-server-20130415-en-us-30GB tom 'xxx'
info:    Executing command vm create
+ Looking up image
+ Looking up cloud service
+ Creating cloud service
+ Retrieving storage accounts
+ Configuring certificate
+ Creating VM
info:    vm create command OK
~~~

You can check the configuration details of your new VM using the following command:

~~~
$ azure --json vm list
~~~

This will output a JSON document containing the details for all your VMs, and the one you just created should look like this:

~~~
{
    "DNSName": "mycdh.cloudapp.net",
    "VMName": "mycdh",
    "IPAddress": "10.0.0.4",
    "InstanceStatus": "ReadyRole",
    "InstanceSize": "Small",
    "InstanceStateDetails": "",
    "OSVersion": "",
    "Image": "b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-12_04_2-LTS-amd64-server-20130415-en-us-30GB",
    "DataDisks": "",
    "Network": {
        "Endpoints": [
            {
                "LocalPort": "22",
                "Name": "ssh",
                "Port": "22",
                "Protocol": "tcp",
                "Vip": "137.117.198.34"
            }
        ]
    }
}
~~~

As you can see, the internal IP address assigned to your VM is indeed `10.0.0.4`, and thus will be used as DNS server by all the VMs we will create next. Now let's turn this VM into an actual DNS!

You can now ssh into your new VM, if your SSH configuration is correct you should be able to log in without a password prompt. Otherwise just make sure you correctly copied your private key file into your `.ssh` directory.

Configuring the DNS using BIND
------------------------------

Now if you look at your DNS configuration, you should see something like this:

~~~
# cat /etc/resolv.conf
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 10.0.0.4
search 52601348a18744ec9cbf2e9c992072da.mycdh.3744114705.europewest.internal.cloudapp.net
~~~

This is fine, however in order to install our DNS we need to be able to resolve names until we have installed BIND on our VM! For now, temporarily comment out 10.0.0.4 and use a public DNS:

~~~
#nameserver 10.0.0.4
nameserver 8.8.8.8
~~~

Now you should have DNS resolution in order to run apt-get and install BIND:

~~~
# apt-get install bind9
# /usr/sbin/named
~~~

Now that BIND is running, you should be able to switch your DNS back to `10.0.0.4` in `/etc/resolv.conf`.

We are going to configure BIND in order to resolve our host names. We don't need a real domain, as this name resolution will only take place within the bounds of our deployment; I am going to use a fake TLD, "internal", and setup the forward and reverse zone files.

You can edit named.conf.local and add the following lines:

~~~
zone "internal" {
        type master;
        file "/etc/bind/db.internal";
};

zone "10.in-addr.arpa" {
        type master;
        file "/etc/bind/db.10";
};
~~~

You then basically copy db.empty as a starting point and add your configuration. This should look like this:

In `db.internal`:

~~~
$TTL    86400
@       IN      SOA     localhost. root.localhost. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns.internal.
ns      IN      A       10.0.0.4
mycdh   IN      A       10.0.0.4
~~~

In `db.10`:

~~~
$TTL    86400
@       IN      SOA     localhost. root.localhost. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns.internal.
4.0.0   IN      PTR     mycdh.internal.
~~~

Now you should run a few tests using `nslookup` to make sure everything is in order, e.g. the forward and reverse resolutions work.

To make BIND re-read its configuration file, find its PID using `ps aux` or in `/var/run/named/named.pid`, and send it a SIGHUP signal, i.e. `kill -HUP (pid)`.

Initializing the cluster
------------------------

We are now ready to create more virtual machines! These will be the actual nodes in our Hadoop cluster, and we will name them with numeric suffixes so that we can identify them easily. We will use a similar command line as above, except we are going to request larger intance sizes.

You can quickly create four instances from the Bash prompt using a mini-script like this:

~~~
$ for i in 1 2 3 4; do
> azure vm create --virtual-network-name cdh --subnet-names Subnet-1 --affinity-group cdh-AG --vm-size large --ssh 22 --ssh-cert ./myCert.pem mycdh-$i b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-12_04_2-LTS-amd64-server-20130415-en-us-30GB tom 'Pass123!'
> done
~~~

This should give you four more machines in a few minutes. You can quickly check the DNS names and IP addresses allocated like this for example:

~~~
$ azure --json vm list | egrep '(DNSName|IPAddress)'
    "DNSName": "mycdh-2.cloudapp.net",
    "IPAddress": "10.0.0.6",
    "DNSName": "mycdh-3.cloudapp.net",
    "IPAddress": "10.0.0.7",
    "DNSName": "mycdh-4.cloudapp.net",
    "IPAddress": "10.0.0.8",
    "DNSName": "mycdh-1.cloudapp.net",
    "IPAddress": "10.0.0.5",
    "DNSName": "mycdh.cloudapp.net",
    "IPAddress": "10.0.0.4",
~~~

These are the name and addresses that we will add to our BIND configuration in order to resolve all the names and addresses.

We are then going to add some additional storage, for my test I will use 150GB drives, however you could use as much as 1TB per disk, and even add several of these disks to each machine if you need even more storage.

~~~
$ for i in 1 2 3 4; do
> azure vm disk attach-new mycdh-$i 150
> done
~~~

Now we are going to switch to the first machine we created, our "admin" machine, to further configure our new VMs. Let's copy our SSH key over so that we can connect to all other VMs from our first one:

~~~
$ scp ~/.ssh/id_rsa mycdh.cloudapp.net:.ssh/
~~~

Once logged on our admin machine, let's add the IP addresses of our nodes to BIND's configuration (use `kill -HUP` again on the `named` process to force BIND to re-read its configuration files).

In `db.internal`:

~~~
mycdh-1 IN      A       10.0.0.5
mycdh-2 IN      A       10.0.0.6
mycdh-3 IN      A       10.0.0.7
mycdh-4 IN      A       10.0.0.8
~~~

In `db.10`:

~~~
5.0.0   IN      PTR     mycdh-1.internal.
6.0.0   IN      PTR     mycdh-2.internal.
7.0.0   IN      PTR     mycdh-3.internal.
8.0.0   IN      PTR     mycdh-4.internal.
~~~

To make things easier, you can also add the following line to `/etc/resolv.conf` in order to make "internal" the default domain searched, so that you don't have to type the complete name each time:

~~~
search internal
~~~

You should now be able to log on to all three other VMs using their FQDN, this will initialize the SSH trust and make sure you can connect without being prompted for a password.

~~~
$ ssh mycdh-1
$ ssh mycdh-2
$ ssh mycdh-3
$ ssh mycdh-4
~~~

Now we have to make it so we can run commands on all nodes as root via `sudo`. We are going to overwrite the configuration created by the Windows Azure Agent at first boot (`/usr/sbin/waagent`) to make sure we can `sudo` without a password. Of course you should replace the account there by the account you created when you ran the `azure vm create` commands. First on our admin machine:

~~~
# echo "tom ALL = (ALL) NOPASSWD: ALL" > /etc/sudoers.d/waagent
~~~

Let's do this on all our nodes:

~~~
$ for i in mycdh-1 mycdh-2 mycdh-3 mycdh-4
> do
> ssh -t $i "sudo bash -c 'echo \"tom ALL = (ALL) NOPASSWD: ALL\" > /etc/sudoers.d/waagent'"
> done
~~~

Now you can run commands as root without ever being prompted for a password. What really happens is that you ssh implicitly using your current user name ("tom" in my case), since it exists on all machines. You can then execute "sudo" via ssh, and pass it a command to execute as root in turn, e.g.:

~~~
$ ssh tomcdh-2 sudo id
~~~

If you get warnings from sudo about not being able to resolve host names, you need to add the "search" option to resolv.conf on the other machines as well:

~~~
$ for i in mycdh-1 mycdh-2 mycdh-3 mycdh-4
> do
> ssh -t $i "sudo bash -c 'echo \"search internal\" >> /etc/resolv.conf'"
> done
~~~

You will then need to partition, format and mount the new disk on every machine. You can easily log in as root using:

~~~
$ ssh -t mycdh-1 sudo -i
~~~

And then use the following shell snippet. It will partition with `sfdisk` the Windows Azure volume that you attached previously using `azure vm disk attach-new`, format it, add it to `fstab`, and finally mount it on `/data`.

~~~
echo "," | sfdisk /dev/sdc
mkfs.ext3 /dev/sdc1
echo "/dev/sdc1 /data ext3 defaults 0 0" >> /etc/fstab
mkdir /data
mount /data
~~~

Do this for all four VMs. Now you should have a formatted disk mounted on `/data` on every of your future nodes.

Preparing for installation
--------------------------

Let's `apt-get update` on our admin VM and all four nodes:

~~~
$ sudo apt-get update
$ for i in mycdh-1 mycdh-2 mycdh-3 mycdh-4; do ssh -t $i sudo apt-get update; done
~~~

I am now looking at the ["Cloudera Manager Free Edition Installation Guide"](http://www.cloudera.com/content/cloudera-content/cloudera-docs/CM4Free/latest/Cloudera-Manager-Free-Edition-Installation-Guide/Cloudera-Manager-Free-Edition-Installation-Guide.html), and more precisely the section "Automated Installation of Cloudera Manager and CDH".

Let's download and run the installer:

~~~
$ wget http://archive.cloudera.com/cm4/installer/latest/cloudera-manager-installer.bin
$ chmod +x ./cloudera-manager-installer.bin
$ sudo ./cloudera-manager-installer.bin
~~~

You should now see the beautiful retro-vintage text-based Cloudera Manager installation screen! Accept (and read) all the licenses, then the installation will proceed and, if everything goes well, you will be presented with the following success screen.

As instructed, we need to connect to the Web application on port 7180. Since we are running on a server VM, we don't have a browser we can launch to open localhost, so let's instead open the port to the outside world:

~~~
$ azure vm endpoint create mycdh 7180
~~~

>Note :this currently fails in the CLI, see [Issue #543](https://github.com/WindowsAzure/azure-sdk-tools-xplat/issues/543). For now we have to use the Portal to create endpoints on VMs that are part of a Virtual Network.

In the Management Portal, go to the Virtual Machines module, and select your admin VM in the list ("mycdh" in my case), then go to the Endpoints tab. You should see a rule fort TCP port 22 (SSH). On the dialog that appears, select Add Endpoint, click Next, then give a name to your new endpoint (e.g. "ClouderaManager"), and enter 7180 for the Public and Private port fields. The new rule should now appear in the list of Endpoints.

Now you can connect to the Cloudera Manager interface using the external URL for your admin machine (e.g. in my case, http://mycdh.cloudapp.net:7180/) and start the Web-based installation wizard.

Using the Cloudera Manager to deploy the Hadoop cluster
-------------------------------------------------------

On the "Thank you" screen, click Continue.

On the "Specify hosts", use the following pattern to describe your hosts:

~~~
mycdh-[1-4].internal
~~~

You need to specify the FQDN, otherwise you will run into problems later when the JobTracker looks up the IP address for validation: the reverse lookup must exactly match the hostname used.

On the first "Cluster Installation" screen, I removed Impala but otherwise I left all the other options by default.

On the "SSH credentials" screen, you will need to check "Another user" and enter the username used to log on your VMs. You will also need to check "All hosts accept same private key" and provide the key we generated above. I run [Cygwin](http://www.cygwin.com/) on my Windows machine and I used this command to copy the key from my Linux workstation in Windows Azure to `C:\TEMP`:

~~~
$ scp tom@tombuntu.cloudapp.net:~/cloudera_beta/PrivateKeyCDH /cygdrive/c/TEMP
~~~

Click Continue and confirm that you are not using an SSH passphrase.

On the "Installation in progress" screen, everything should install successfully. If you encounter SSH credentials issue, this screen will help you troubleshoot the problem.

The next screen will install all the selected parcels to your machines.

On the "Host inspection" screen, everything should be **green**! Pay special attention to host name resolution warnings at this stage, they may cause problems later if they are not resolved now.

On the "CDH4 services" screen, choose the services you want to install. I will choose "Core with Real-Time Delivery" for a base Hadoop stack plus HBase. If you want to fine-tune which services will run on which hosts, you can click on "Inspect Role Assignments" to check what will run where. You will see that the first machine gets a larger share of services, and in production you might want to make it a larger instance size.

On the "Database setup" screen I kept the Embedded Database, then clicked on Test Connection, then Continue.

When you reach the "Review configuration changes" page, remove the `/mnt/resource` disk from all nodes running the HDFS service (this is a local, non-persistent disk and we don't want to store data on it). You will also need to lower the tolerance accordingly (i.e. set it to 0 in this case). Likewise for the NameNode and MapReduce services, remove all occurences of `/mnt/resource`.

**That's it!** All the services should start successfully. Congratulations!

Accessing the Hue Web UI
------------------------

Now you may want to access the nice Web UI that Cloudera integrates in their distribution. On the main Services page, click on "hue1" and you will see a status page where you will find a "Hue Web UI" button. Press it, and it should fail! The management portal redirects you to the internal address of the Hue node, e.g. `http://mycdh-1.internal:8888/`, and this address is not known from the outside world. You should use the corresponding public address, i.e. `mycdh-1.cloudapp.net` in my case, and open the port (8888) by creating an endpoint in the Windows Azure Management Portal. You can proceed in the same manner whenever you need to access a service directly on one of the nodes.

Annex 1: Manually creating a Virtual Network
--------------------------------------------

Connect to your Windows Azure management portal, and select New, Networks, Virtual Network, Quick Create, and just select a name and a location (data center) for your Virtual Network; all our VMs will later be created in the data center you select. You can also select the size and the address prefix, I will just go for the defaults.

Once the Virtual Network is created, click on its name and then on Dashboard to see how it was configured. You will see that an Affinity Group was automatically created, for example if you named your Virtual Network "cdh" if will be associated with a "cdh-AG" Affinity Group. Now if you click on Configure, you will find another important piece of information: a Subnet was automatically created for you, typically named "Subnet-1" if you went for the Quick Create defaults. We will need these two parameters later to create our Virtual Machines.

Annex 2: Installation the Windows Azure command-line tools for Mac/Linux
------------------------------------------------------------------------

First install Node.JS, don't forget to check on the [Node.JS downloads](http://nodejs.org/download/) page what is the latest version!

~~~
$ sudo apt-get update
$ sudo apt-get install build-essential
$ wget http://nodejs.org/dist/v0.10.5/node-v0.10.5.tar.gz
$ tar xf node-v0.10.5.tar.gz
$ cd node-v0.10.5/
$ ./configure
$ sudo make install
~~~

Then install the Windows Azure CLI using `npm`:

~~~
$ sudo npm install azure-cli -g
~~~

Then initialize the Windows Azure CLI:

~~~
$ azure account import Azure-credentials.publishsettings
~~~
