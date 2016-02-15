---
layout: post
date: 2014-06-18
title: Building your Ubuntu PHP development box on Azure using Vagrant
---

With Azure you can easily create and use a development environment in the Cloud. You can create a Virtual Machine from the [Management Portal](https://manage.windowsazure.com/) or from the [command line](http://azure.microsoft.com/en-us/documentation/articles/xplat-cli/), but if you want more automation, you can use a tool like [Vagrant](http://www.vagrantup.com/) to automatically start and configure your Virtual Machine so that it is ready to use, instead of following manual installation steps to configure your environment.

Our friends at MS Open Tech have recently published a [Vagrant Azure Provider](https://github.com/MSOpenTech/Vagrant-Azure) that allows Vagrant to control and provision machines in Windows Azure. I would like to show you how to use this provider to start a VM and then automatically provision it with a development environment suitable for PHP/LAMP and [Symfony2](http://symfony.com/) development.

First, you should [download and install Vagrant for Windows](http://www.vagrantup.com/downloads.html). Then, installing the Azure plugin is as simple as:

	vagrant plugin install vagrant-azure

Then we are going to base our configuration on the "dummy Azure box" provided by MS Open Tech:

	vagrant box add azuredummybox https://github.com/msopentech/vagrant-azure/raw/master/dummy.box

Next, create an empty directory as your project root, and create a Vagrantfile in it. Here is a Vagrantfile for Azure, let's look at it in detail.

~~~ruby
Vagrant.configure('2') do |config|
    config.vm.box = 'trusty64'

    config.vm.provider :azure do |azure, override|
        override.vm.box = 'azuredummybox'
        override.ssh.username = 'azureuser' # assigned below
        override.ssh.password = 'Pass123!' # assigned below

        azure.mgmt_certificate = 'C:\Dev\management_cert.pfx'
        azure.mgmt_endpoint = 'https://management.core.windows.net'
        azure.subscription_id = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
        azure.storage_acct_name = 'mystorage' # optional. A new one will be generated if not provided.

        # Ubuntu Precise 12.04 LTS
        #azure.vm_image = 'b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-12_04_4-LTS-amd64-server-20140428-en-us-30GB'
        # Ubuntu Trusty 14.04 LTS
        azure.vm_image = 'b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-14_04-LTS-amd64-server-20140416.1-en-us-30GB'

        azure.vm_user = 'azureuser' # defaults to 'vagrant' if not provided
        azure.vm_password = 'Pass123!' # min 8 characters. should contain a lower case letter, an uppercase letter, a number and a special character

        azure.vm_name = 'phpdevbox' # max 15 characters. contains letters, number and hyphens. can start with letters and can end with letters and numbers
        azure.cloud_service_name = 'phpdevbox' # same as vm_name. leave blank to auto-generate
        #azure.deployment_name = '' # defaults to cloud_service_name
        azure.vm_location = 'West Europe' # e.g., West US
        azure.ssh_private_key_file = 'C:\Dev\id_rsa'
        azure.ssh_certificate_file = 'C:\Dev\myCert.pem'

        # Provide the following values if creating a *Nix VM
        azure.ssh_port = '22'

        # Open HTTP port
        azure.tcp_endpoints = '80,8000'
    end

    config.vm.provision :shell, :path => "bootstrap.sh"
end
~~~

Here are the parameters you need to change:

- `override.ssh.username` should be the same as azure.vm_user: the login that Vagrant will use to connect via ssh.
- `override.ssh.password` and `azure.vm_password`: the password.
- `azure.mgmt_certificate`: you need to point Vagrant to your Azure management certificate. There are a couple ways to obtain it, the simplest is to [extract it from your Publish Settings file](https://github.com/Azure/azure-content/blob/master/articles/linux-create-management-cert.md).
- `azure.subscription_id`: this is your Subscription ID, you will find it in your Publish Settings file as well.
- `azure.vm_image`: the name of the image you want to use; I included two examples for Ubuntu Precise Pangolin and Trusty Tahr, but you should double-check for the latest versions, for example using `azure vm image list` from the CLI tools.
- `azure.vm_name`: the host name for your VM.
- `azure.cloud_service_name`: this will be the public name for your VM; preferably the same as the host name.
- `azure.vm_location`: the name of the data center you want to use; `azure vm location list` to retrieve a list of locations.
- `azure.ssh_private_key_file` and `azure.ssh_certificate_file` are used to configure your VM's ssh access. We have a tutorial explaining how to [generate Windows Azure compatible keys for Linux](http://azure.microsoft.com/en-us/documentation/articles/linux-use-ssh-key/).
- `azure.tcp_endpoints`: let's open both the standard HTTP port (80) and port 8000 in order to run the PHP built-in Web server from userspace.

So far, we have configured everything we need to create and provision the Virtual Machine, now we need to configure the development environment for LAMP, i.e. install PHP, MySQL, etc. An easy way to do it is to instruct Vagrant to run a shell script to bootstrap the VM, this is what the following line does:

~~~ruby
config.vm.provision :shell, :path => "bootstrap.sh"
~~~

This will automatically run the `bootstrap.sh` script in the VM. Let's have a look at the script.

~~~bash
#!/bin/bash

# Add Node repo
add-apt-repository ppa:chris-lea/node.js

apt-get update

# LAMP etc.

# MySQL password
debconf-set-selections <<< 'mysql-server mysql-server/root_password password Pass123!'
debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password Pass123!'

# LAMP
apt-get -q -y install lamp-server^

# Misc PHP stuff
apt-get -q -y install php-apc php5-xdebug php5-intl

# Git
apt-get -q -y install git

# Node
apt-get install -q -y python-software-properties python g++ make nodejs

# Azure CLI
npm install azure-cli -g

# PHP configuration

for f in /etc/php5/cli/php.ini /etc/php5/apache2/php.ini
do
  sed -i "s/;date.timezone =/date.timezone = Europe\/Paris/" $f
  echo "xdebug.max_nesting_level=250" >> $f
done
~~~

This script basically installs everything you need to run a LAMP application, plus the Azure Command-Line Interface and configures some Symfony2-specific parameters. In more detail:

- Add [Chris Lea's repository](https://chrislea.com/2013/03/15/upgrading-from-node-js-0-8-x-to-0-10-0-from-my-ppa/) to facilitate installing a recent version of Node.
- Update the repository database.
- Pre-configure a root password for MySQL.
- Install the magic LAMP server bundle.
- Install Git.
- Install Node.JS and a few required packages.
- Configures a couple of recommended parameters for Symfony2 in the php.ini.

Now all you need to do to start your development environment is to run the magic Vagrant commands to start and provision the VM:

~~~
vagrant up --provider=azure
vagrant provision
~~~

The machine is now ready to use! You can log in using the following command:

~~~
vagrant ssh
~~~

Now that your development machine environment is entirely automated, you can check it into your source code repository and keep track of any changes you need to make to it. Other developers can use this configuration in order to exactly reproduce the same development environment, automatically.

Let's test our new environment, for example we can check out a sample Symfony2 project:

~~~
git clone https://github.com/tomconte/symfony-azure.git
~~~

Run Composer:

~~~
cd symfony-azure
curl -sS https://getcomposer.org/installer | php
php composer.phar install
~~~

Symfony will prompt you for your configuration information; you can accept all the defaults, but don't forget to give the right root password for MySQL! Now you should be able to check everything is OK and run the demo app:

~~~
php app/check.php
php app/console server:run 0.0.0.0:8000
~~~

I hope this article gave you a good idea of how to use Vagrant together with Azure to easily spin up automated and standardized development environments.

Below I have included a Gist with the complete scripts and configuration files mentioned in the article.

<script src="https://gist.github.com/tomconte/e7032f265b0b3124a7ce.js">
</script>
