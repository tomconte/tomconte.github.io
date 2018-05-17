---
title: "Installing SUSE CAP on AKS"
layout: post
---

This article describes the steps to deploy the SUSE Cloud Application Platform (CAP) on Azure Kubernetes Service (AKS). SUSE Cloud Application Platform is a software platform for cloud-native application deployment based on SUSE Cloud Foundry; SUSE Cloud Foundry differentiates itself from other Cloud Foundry distributions by running in Linux containers managed by Kubernetes, rather than virtual machines managed with BOSH, for greater fault tolerance and lower memory use.

Please first review the [SUSE Cloud Application Platform documentation](https://www.suse.com/documentation/cloud-application-platform-1/). These instructions are only additional or alternate steps you can take to prepare the Azure environment to deploy CAP. You should read the rest of the CAP documentation to get full instructions.

Following the steps below, you will configure an Azure load balancer associated to a public IP address, so that the traffic to the CAP services can be load-balanced over all the Kubernetes nodes. The SUSE CAP documentation describes another, simpler setup, where the public IP address is directly attached to one of the Kubernetes nodes.

This guide was tested with the "minimal installation for testing" configuration described in the documentation.

## Create an AKS cluster

First create a resource group:

    az group create --name cap --location westeurope

NOTE: make sure you create the Resource Group in a region where AKS is available; see the [AKS documentation "region availability"](https://docs.microsoft.com/en-us/azure/aks/container-service-quotas#region-availability) section.

Then create an AKS instance:

    az aks create --resource-group cap --name cap --node-vm-size Standard_DS2_v2

NOTE: see the [`aks create` command documenation](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az-aks-create) if you want to change AKS defaults, like the administration user name, SSH key, number of nodes, etc.

Make sure you provision enough nodes and that you select an appropriate VM size for your workload. In our example we used the default number of nodes (3) but selected a slightly larger instance size.

Once the deployment is finished, retrieve the `kubectl` credentials:

    az aks get-credentials --resource-group cap --name cap

You can now check that you can connect to your Kubernetes cluster:

    kubectl get pods --all-namespaces

## Identify the AKS supplemental resource group

Once AKS has finished deploying, you will find that it automatically created a second resource group with a name in the form _MC_myResourceGRoup_myAKSCluster_location_. You can find this resource group using the `az group` command:

    az group list -o table | grep MC_

Set a variable in your shell session so you can refer to it later, for example:

    export RGNAME=MC_cap_cap_westeurope

## Enable swap accounting

Cloud Foundry requires swap accounting to be turned on in the kernel, which is not the default in AKS nodes. The following script will use `az` commands to modify the GRUB configuration on each node and reboot the VM. You will need the `jq` command to be installed to run the script.

``` sh
vmnodes=$(az vm list -g $RGNAME | jq -r '.[] | select (.tags.poolName | contains("node")) | .name')

for i in $vmnodes
do
   az vm run-command invoke -g $RGNAME -n $i --command-id RunShellScript --scripts "sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT=\"console=tty1 console=ttyS0 earlyprintk=ttyS0 rootdelay=300\"/GRUB_CMDLINE_LINUX_DEFAULT=\"console=tty1 console=ttyS0 earlyprintk=ttyS0 rootdelay=300 swapaccount=1\"/g' /etc/default/grub.d/50-cloudimg-settings.cfg"
done

for i in $vmnodes
do
   az vm run-command invoke -g $RGNAME -n $i --command-id RunShellScript --scripts "sudo update-grub"
done

for i in $vmnodes
do
   az vm restart -g $RGNAME -n $i
done
```

## Create and configure a Public IP and Load Balancer

CAP needs a public endpoint to be exposed so you can access the services.

First create a static public IP address:

    az network public-ip create --resource-group $RGNAME --name cap-public-ip --allocation-method Static

Then create a load balancer, using the static IP address we just created:

    az network lb create --resource-group $RGNAME \
    --name cap-lb \
    --public-ip-address cap-public-ip \
    --frontend-ip-name cap-lb-front \
    --backend-pool-name cap-lb-back

Now, we need to add the VM network interface cards (NICs) to the load balancer's back-end pool. The following script will iterate over all the NICs in the resource group, and add them to the `cap-lb-back` pool.

``` sh
nicnames=$(az network nic list --resource-group $RGNAME | jq -r '.[].name')

for i in $nicnames
do
    az network nic ip-config address-pool add \
    --resource-group $RGNAME \
    --nic-name $i \
    --ip-config-name ipconfig1 \
    --lb-name cap-lb \
    --address-pool cap-lb-back
done
```

## Configure load balancing and network security rules

We need to add the load balancing rules to allow traffic to be routed for all the ports that CAP needs. The following script will create the rules and associated probes.

``` sh
for i in 80 443 4443 2222 2793
do
    az network lb probe create --resource-group $RGNAME \
    --lb-name cap-lb \
    --name probe-$i \
    --protocol tcp \
    --port $i

    az network lb rule create --resource-group $RGNAME \
    --lb-name cap-lb \
    --name rule-$i \
    --protocol Tcp \
    --frontend-ip-name cap-lb-front \
    --backend-pool-name cap-lb-back \
    --frontend-port $i \
    --backend-port $i \
    --probe probe-$i
done
```

Finally, we need to modify the Network Security Group (NSG) that AKS created, in order to allow traffic to the different ports. The following script will create the rules.

``` sh
nsg=$(az network nsg list --resource-group=$RGNAME | jq -r '.[].name')
pri=200

for i in 80 443 4443 2222 2793
do
    az network nsg rule create --resource-group $RGNAME \
    --priority $pri \
    --nsg-name $nsg \
    --name cap-$i \
    --direction Inbound \
    --destination-port-ranges $i \
    --access Allow

    pri=$(expr $pri + 1)
done
```

## Deploy CAP using Helm

You can now proceed to the instructions to [deploy CAP using Helm](https://www.suse.com/documentation/cloud-application-platform-1/book_cap_deployment/data/sec_cap_helm-deploy.html). Here are some notes about changes you will need to make in the `scf-config-values.yaml` configuration file.

- `storage_class`: use either `default` or `managed-premium`, depending on your AKS cluster configuration. You can list the available storage classes using `kubectl get storageclasses`.
- `auth`: use `none` as AKS does not currently support RBAC.
- `external_ips`: you will need to list the **internal** IP addresses that were allocated to the VM NICs. Use the command below to list the internal IP addresses:

``` sh
az network nic list --resource-group $RGNAME | jq -r '.[].ipConfigurations[].privateIpAddress'
```

- You will need to configure your DNS to point to the public IP address of the cluster. Use the command below to retrieve the public IP address:

``` sh
az network public-ip show --resource-group $RGNAME --name cap-public-ip --query ipAddress
```

You should now be able to successfully deploy CAP on top of AKS using the Helm charts provided by SUSE.

## Deployment tips

Once the UAA service is deployed, you should be able to access it using the DNS name listed in the configuration file. For example:

    curl -k https://uaa.cap.aks.ninja:2793/.well-known/openid-configuration

This should return a JSON object. You should make sure this service works before attempting to deploy the main SCF chart.

## Conclusion

Hopefully the steps described in this article demonstrate that Azure Kubernetes Service is a great option to easily deploy and scale the SUSE Cloud Application Platform in the cloud.
