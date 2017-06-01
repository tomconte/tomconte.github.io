---
title: "Azure Container Service: Docker swarm mode + NFS server deployment"
layout: post
---

This article details how to create a Docker swarm mode cluster using the Azure Container Service engine, alongside a separate Virtual Machine acting as an NFS server, then mount an NFS filesystem from a service running on the swarm.

## Prepare the resources

We are going to create two resource groups, one for the shared resources (VNet and NFS server) and another one for the Swarm cluster. This way we can cleanly separate the two parts of the infrastructure.

```
az group create -n tco-swarm-shared -l northeurope
az group create -n tco-swarm-cluster -l northeurope
```

Next, we are going to create the VNet. There are several ways to do it, I am going to use the Azure command-line interface, but we could also use an ARM template.

```
az network vnet create -n swarm-vnet -g tco-swarm-shared --address-prefixes 10.0.0.0/16
```

Then add three subnets:

```
az network vnet subnet create -n nfs-subnet -g tco-swarm-shared \
--vnet-name swarm-vnet --address-prefix 10.0.0.0/24
az network vnet subnet create -n master-subnet -g tco-swarm-shared \
--vnet-name swarm-vnet --address-prefix 10.0.1.0/24
az network vnet subnet create -n agent-subnet -g tco-swarm-shared \
--vnet-name swarm-vnet --address-prefix 10.0.2.0/24
```

From the output, note the ID of the master-subnet and agent-subnet that we will use to deploy Docker via ACS; if you miss them you can use the following command:

```
az network vnet list -g tco-swarm-shared
```

This will display all the VNet details, including the various resources IDs; the ones for the subnets should look like this:

```
/subscriptions/252281c3-8a06-4af8-8f3f-d6af13e4fde3/resourceGroups/tco-swarm-shared/providers/Microsoft.Network/virtualNetworks/swarm-vnet/subnets/master-subnet
/subscriptions/252281c3-8a06-4af8-8f3f-d6af13e4fde3/resourceGroups/tco-swarm-shared/providers/Microsoft.Network/virtualNetworks/swarm-vnet/subnets/agent-subnet
```

## Create a VM for the NFS server

We are going to create a VM in the nfs-subnet where we will run the NFS server. The command below create an Ubuntu VM in the proper VNet/subnet, using SSH key authentication, and no public IP address.

```
az vm create -n nfs-server -g tco-swarm-shared \
--image UbuntuLTS --public-ip-address "" \
--size Standard_DS2_v2 --vnet-name swarm-vnet --subnet nfs-subnet \
--admin-username azureuser --ssh-key-value ~/.ssh/id_rsa.pub
```

## Crate the ACS Swarm Mode cluster

We are going to use [ACS-Engine](https://github.com/Azure/acs-engine) to configure a custom ACS cluster that will use Docker in swarm mode and deploy into our pre-existing subnets.

We will need to use the subnets IDs in our ACS-Engine configuration to tell it where to deploy the nodes; you can first look at [an example in the ACS-Enginer repo](https://github.com/Azure/acs-engine/blob/master/examples/vnet/swarmmodevnet.json) to see how to configure the ACS cluster to use our VNet.

We are going to use a simpler configuration, based on the basic example `swarmmode.json`; we basically need to add the `vnetSubnetId` elements to specify which subnet we want the master and agent nodes to be deployed in.

```json
{
  "apiVersion": "vlabs",
  "properties": {
    "orchestratorProfile": {
      "orchestratorType": "SwarmMode"
    },
    "masterProfile": {
      "count": 3,
      "dnsPrefix": "tco-swarm-master",
      "vmSize": "Standard_D2_v2",
      "vnetSubnetId": "/subscriptions/252281c3-8a06-4af8-8f3f-d6af13e4fde3/resourceGroups/tco-swarm-shared/providers/Microsoft.Network/virtualNetworks/swarm-vnet/subnets/master-subnet",
      "firstConsecutiveStaticIP": "10.0.1.5" 
    },
    "agentPoolProfiles": [
      {
        "name": "agentpublic",
        "count": 3,
        "vmSize": "Standard_D2_v2",
        "dnsPrefix": "tco-swarm-agent",
	"vnetSubnetId": "/subscriptions/252281c3-8a06-4af8-8f3f-d6af13e4fde3/resourceGroups/tco-swarm-shared/providers/Microsoft.Network/virtualNetworks/swarm-vnet/subnets/agent-subnet",
        "ports": [
          80,
          443,
          8080
        ]
      }
    ],
    "linuxProfile": {
      "adminUsername": "azureuser",
      "ssh": {
        "publicKeys": [
          {
            "keyData": "ssh-rsa xxxx"
          }
        ]
      }
    }
  }
}

```

We then use ACS-Engine to generate the deployment template. You will need to [run the build container as described in the documentation](https://github.com/Azure/acs-engine/blob/master/docs/acsengine.md) and build the `acs-engine` command first; then from inside the container:

```
./acs-engine generate swarmmode.json
```

This will create a bunch of JSON files in the `_output` folder; they constitute the ARM template that will deploy the Swarm Mode cluster. Using this template, we can now deploy the swarm cluster into the second Resource Group we created at the beginning:

```
az group deployment create -g tco-swarm-cluster \
--template-file ./_output/tco-swarm-master/azuredeploy.json \
--parameters @./_output/tco-swarm-master/azuredeploy.parameters.json
```

The nodes will deploy into the same VNet and they will be able to reach the NFS server in its own subnet.

## Testing the setup

This is a set of notes showing how to check that the swarm workers are able to access the NFS filesystem. There are certainly several ways to do this and these are only intended as manual testing instructions!

You can connect via SSH to one of the Docker swarm master nodes, and then rebound from there to the other machines in the cluster. You can use the internal DNS configured by default to connect to the NFS server: (you will need to copy your private SSH key over)

```
ssh nfs-server
```

Install NFS and create a directory to export:

```
sudo apt-get install nfs-kernel-server
sudo mkdir -p /export/www
sudo chown -R www-data:www-data /export/www
```

Add a line to `/etc/exports`:

```
/export/www     *(rw,sync,no_subtree_check)
```

Export the share:

```
sudo exportfs -ra
```

On the client side (Swarm cluster nodes), you will first need to install the NFS client:

```
sudo apt-get install nfs-common
```

You can then test that you can mount the NFS share:

```
sudo mkdir -p /www
sudo mount -t nfs nfs-server:/export/www /www
```

(you will need to run these steps on every node in the cluster; this could be automated as part of the ARM template deployment or through any other automation mechanism, e.g. Ansible, etc.)

Deploy a service on the swarm, mounting the volume:

```
docker service create --name some-nginx \
--mount type=bind,source=/www,destination=/usr/share/nginx/html,ro \
--publish 80:80 -d nginx
```

And scale it:

```
docker service scale some-nginx=3
```

You can now access the public IP address for the swarm cluster using a browser and check that you can see the content.
