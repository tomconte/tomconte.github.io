---
title: "Azure VM Custom Script Extensions with Terraform"
layout: post
---

Terraform provides support for Azure Virtual Machine Custom Script extensions, that are often used to configure a newly created virtual machine and prepare it so it is ready to perform its role. Typical tasks performed in these custom scripts include installing additional packages, configuring system services, creating users, etc.

In very simple cases, a custom script can be defined as a simple `commandToExecute`, for example if all is needed is to install software packages. However in most cases, there is a need to run a more complex script to perform the configuration. You can specify a script to download from a network location, using `fileUris`, however that creates a dependency to an external component, and that can lead to some issues, for example if the network location is unreachable, or if you want to find out exactly which version of the script was used and executed during a past deployment.

As described in the [Custom Script Extension](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-linux#extension-schema) documentation, you can also pass the whole script within the extension settings, as a Base64-encoded string in the `script` property. This is a nice solution where the script is included within the deployment, with no external dependencies.

Terraform provides a couple ways to assemble this `script` property in a very easy way.

The simplest one is to use the `filebase64` function, that will read in a file, encode it with Base64, and include the result in the resource configuration.

```
resource "azurerm_virtual_machine_extension" "test" {
  name                 = "hostname"
  location             = "${azurerm_resource_group.test.location}"
  resource_group_name  = "${azurerm_resource_group.test.name}"
  virtual_machine_name = "${azurerm_virtual_machine.test.name}"
  publisher            = "Microsoft.Azure.Extensions"
  type                 = "CustomScript"
  type_handler_version = "2.0"

  settings = <<SETTINGS
    {
        "script": "${filebase64("custom_script.sh")}"
    }
SETTINGS
}
```

Another option, if you need to pass in some values to the script, is to use the new `templatefile` function, available in Terraform 0.12 and later. This lets you read the contents of the file, and render it as a template using a set of supplied variables. Using this function you can pass any variable from the Terraform configuration to the script using the usual `${ ... }` sequences. You will also have to use the `base64encode` function to explicitly Base64-encode the result.

```
resource "azurerm_virtual_machine_extension" "test" {
  name                 = "hostname"
  location             = "${azurerm_resource_group.test.location}"
  resource_group_name  = "${azurerm_resource_group.test.name}"
  virtual_machine_name = "${azurerm_virtual_machine.test.name}"
  publisher            = "Microsoft.Azure.Extensions"
  type                 = "CustomScript"
  type_handler_version = "2.0"

  settings = <<SETTINGS
    {
        "script": "${base64encode(templatefile("custom_script.sh", {
          vmname="${azurerm_virtual_machine.test.name}"
        }))}"
    }
SETTINGS
}
```

In this example we are passing a `vmname` variable, initialized using the value for `azurerm_virtual_machine.test.name`. This variable can be used in the script using the syntax below:

```
#!/bin/sh

echo "This is a test script."

echo "This machine name is: ${vmname}"
```

I find this a very clean and simple way to keep the configuration scripts separated from the Terraform configuration, but still end up with a complete resource definition with no external dependencies.
