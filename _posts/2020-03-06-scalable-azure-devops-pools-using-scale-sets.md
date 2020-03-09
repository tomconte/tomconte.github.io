---
title: "Scalable Azure DevOps Agent Pools using VM Scale Sets"
layout: post
---

When using [Azure DevOps Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/), you can get started very quickly by using [Microsoft-hosted build agents](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops) to run your build and test jobs. However there are some cases where custom (a.k.a. "self-hosted") build agents might be desirable, the typical case being that you need to install a bunch of tools that are not present in the Microsoft-hosted agents. It is always possible to install custom tools on the fly, and it mostly works when all you need is a couple dependencies, however this method will not be practical when you need a full toolchain installed. 

Self-hosting build agents is very simple, and all you need to do is install and run the [Azure DevOps Build Agent](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux?view=azure-devops) anywhere you want. As long as the agent has Internet connectivity, it will be able to receive and run Pipeline jobs. Typically, you could create a bunch of Azure Virtual Machines and run the agent there. But what if you want to optimize your costs by scaling the agent pool up and down? The typical scenario for this would be that you will need to run hundreds or thousands of builds during a typical work day, but the activity is almost nonexistant during the night. You could save a lot by scaling down your build pools during the night, and scaling them up again in the morning. This is typically a scenario where [Virtual Machine Scale Sets](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview) (VMSS) can be used.

**Nota bene**: the [Azure virtual machine scale set agents](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/vmss?view=azure-devops) feature is currently in preview and might solve this problem for you, without having to use the solution described in this article.

So here is what we are going to build:

- A custom Virtual Machine image that will contain all the tools needed to run the build and test jobs, plus of course the Azure DevOps build agent. I will use [Packer](https://packer.io/) for that task.
- A Virtual Machine Scale Set definition that will instantiate a cluster of Virtual Machines based on our custom image, each of them connecting to the Azure DevOps Build Pool so that we can run jobs on them. I will use an [ARM Template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/) to do this.

## Building the VM image

The VM Image is the basis for our Agent Pool. It should include all the tools needed to build and test your application, plus of course the Azure DevOps Agent so that the VM can join a build pool. I will use [Packer](https://packer.io/) to automate the image creation.

The [Packer configuration](https://github.com/tomconte/vmss-build-pool/blob/master/agent-image.json) is a simple JSON file that describes how you want to build image: which base OS image to use, and what commands to run to prepare it. In my case I will start with an Ubuntu 18.04-LTS image provided by Azure, and I will use an installation script called [`install-build-agent.sh`](https://github.com/tomconte/vmss-build-pool/blob/master/install-build-agent.sh) to install and configure the image. In my case the script installs the following:

- The `build-essential` meta package that includes essential developments tools.
- Docker CE, so I can build and run Docker containers on my agent.
- The Azure CLI, so I can interact with Azure services.
- Any other tools you might need, for example in my case, the Bazel build tool.

The Packer configuration then installs (but does not execute) another script, [`start-agent.sh`](https://github.com/tomconte/vmss-build-pool/blob/master/start-agent.sh), on the VM. That script will be executed at the time a VM is created based on the image, and its main task is to download, install and register the Azure DevOps Agent, so that the VM appears in a build pool and can receive jobs to execute. Any last minute configuration steps can also be performed in that script, for example in my case I an pre-pulling some Docker images so that running the corresponding containers is faster the first time.

Before you can run Packer, you will need to provide some credentials following one of the methods described in the [Packer documentation](https://packer.io/docs/builders/azure.html). In my example I am using the Service Principal method, and your are expected to provide the following environment variables so that Packer can access Azure:

```
export ARM_CLIENT_ID=
export ARM_CLIENT_SECRET=
export ARM_SUBSCRIPTION_ID=
export ARM_TENANT_ID=
```

The image is created by using `packer build`. Packer will instantiate a temporary VM, and will run the installation scripts and perform the other tasks following the configuration. It will then stop and capture the temporary VM, thus creating a VM Image. The name of the target image and resource group names can be modified in the JSON configuration file:

```
"managed_image_name": "build-agent-image",
"managed_image_resource_group_name": "vmss-build-pool",
```

Once the image is built, you are ready to create a VM Scale Set.

## Deploying the Scale Set

According to the [documentation for Virtual Machine Scale Sets (VMSS)](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview), *"Azure virtual machine scale sets let you create and manage a group of identical, load balanced VMs. The number of VM instances can automatically increase or decrease in response to demand or a defined schedule. Scale sets provide high availability to your applications, and allow you to centrally manage, configure, and update a large number of VMs."*

Now that we have a VM Image, we can deploy a VM Scale Set based on that image. The [`azuredeploy.json`](https://github.com/tomconte/vmss-build-pool/blob/master/azuredeploy.json) file contains an [ARM Template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) that will deploy a Scale Set, passing some parameters that the `start-agent.sh` script described above needs in order to configure the DevOps Agent. The script is executed on each node of the Scale Set by using the `extensionProfile` element where a [Custom Script Extension](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-linux) can be configured.

Before you can deploy the template, you will need to edit `deploy-parameters.json` to change the following deployment parameters according to your environment:

- `adminUsername`: the name for the administrative user account to be created on the VM.
- `adminPassword`: the password for the administrative user account.
- `subnetId`: the resource ID of the Virtual Network subnet where the nodes will be created. You will need to create the Virtual Network and the subnet before deploying this template, using any method you prefer (Web Portal, CLI, etc.)
- `managedImageResourceUri`: the resource ID of the custom VM image to use for the nodes. It is displayed in the Packer log after the image creation process is finished.
- `agentOrgUrl`: the URL of the Azure DevOps organization.
- `agentPat`: the Personal Access Token (PAT) used to configure the agent. Please read the [Azure DevOps documentation about Personal Access Tokens](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=preview-page).
- `agentPool`: the name of the Agent Pool where the agent should register.

You can also look at the following element in xxx to modify the capacity of the Scale Set (i.e. number of nodes) and the VM SKU to use (i.e. the type and size of the VM). As you can see, in my example I am deploying two *Standard_F4s_v2* nodes.

```
"sku": {
    "name": "Standard_F4s_v2",
    "capacity": 2
}
```

You can then deploy the template, for example using the Azure CLI:

```
az group deployment create -g vmss-build-pool \
  --template-file azuredeploy.json \
  --parameters @deploy-parameters.json
```

Once the template is deployed, you should see new build agents appear in the pool that you selected. If you scale up the Scale Set, even more agents will be added.

**Caveat:** if you scale down the Scale Set, the agents will become inactive (since the nodes will be deleted), but they will still be visible in the pool. I have not yet implemented a way to remove the node from the DevOps pool on scale down.

## Conclusion

You will find all the source code in the following GitHub repo: [https://github.com/tomconte/vmss-build-pool](https://github.com/tomconte/vmss-build-pool)

One additional configuration that could be added to the VMSS template is a set of [autoscaling rules](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview) so that the number of nodes will automatically scale up and down based on a set of metrics, for example CPU utilization. 
