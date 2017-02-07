---
title: Adding a data disk to a VM using Packer
layout: post
---

Packer is an amazing tool to generate virtual machine images for all your favorite hypervisors, container engines, or cloud platforms. In my case, I have several partners who use Packer to generate images for publishing within the Azure Marketplace.

Recently one of them asked me if Packer could be used to add a data disk to the virtual machine at build time. This is unusual because Azure VM images usually consist of a single VHD file, but the Marketplace also allows you to list additional data disks for your offer if you want. My partner was looking for a way to use Packer in this context, e.g. prepare both the VM image and an additional VHD file for the data disk.

Packer itself does not handle this scenario, however it is fairly easy to implement using out-of-the box provisioners, with a little help from the Azure command-line tool.

Basically you want something like this:

``` json
  "provisioners": [
    {
      "type": "shell-local",
      "command": "add-disk.cmd"
    },
    {
      "type": "shell",
      "script": "init.sh"
    }
  ]
```

The first provisioner, running locally, will use the Azure CLI to add a data disk to the VM. The second provisioner is your usual remote shell script that will be used to prepare the image, i.e. install software, etc.

The only issue is that in order to add a data disk to the temporary VM that Packer uses to build an image, using the `az vm disk attach-new` command, you need to know the name for that VM as well as the name of the Resource Group it was created in, and by default the Azure builder for Packer will generate random names for both.

I sent a [Pull Request to the Packer repo](https://github.com/mitchellh/packer/pull/4468) to add two new configuration variables to the Azure builder: `temp_compute_name` and `temp_resource_group_name`. These configuration variables allow you to choose the names for both elements, which you can then reuse in your local script.

At the moment the PR has been merged but the changes have not been released yet, so you might have to get Packer from the GitHub master branch.

You can add them to your Packer JSON file:

``` json
    "temp_compute_name": "tmppackervm",
    "temp_resource_group_name": "tmppackerrg",
```

And now you can reuse these names in your local script, the one that will add the data disk:

```
az vm disk attach-new --vm-name tmppackervm --resource-group tmppackerrg --vhd https://tcopackersa.blob.core.windows.net/disks/data-disk-2601.vhd --lun 0
```

Of course this is still a bit of hack since the generated data disk VHD will stick around after Packer cleans up the temporary resources; ideally you would add another step to rename the VHD once the image has been created, so that you don't get an error the next time you run the script.

However you can now use this VHD to attach a data disk to an Azure Marketplace VM image.
