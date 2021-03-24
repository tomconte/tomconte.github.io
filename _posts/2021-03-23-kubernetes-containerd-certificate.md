---
title: "Adding a trusted certificate for containerd on Kubernetes using a DaemonSet"
layout: post
---

The Kubernetes project is currently in the process of [migrating its container runtime](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/) from Docker to `containerd`, and is planning to obsolete Docker as a container runtime after version 1.20. In most cases, this should be fairly transparent, but if you click through to the [Dockershim Deprecation FAQ](https://kubernetes.io/blog/2020/12/02/dockershim-faq/), you can see a pretty long list of possible ways this could impact you. 

Recently I looked at a situation that is directly impacted by the migration to `containerd`: what if you are using a private registry, protected by a self-signed certificate? Of course you should not do this, but in practice it seems to be quite popular. In order to allow this, you need to add the self-signed certificate to a trusted list of certificates on the client, i.e. your Kubernetes nodes.

On Kubernetes pre-1.20, which uses the Docker runtime, one popular solution was to use a DaemonSet, that would install the certificate in the Docker configuration on the node, using volume mounts. You can find [StackOverflow discussions](https://stackoverflow.com/questions/53545732/how-do-i-access-a-private-docker-registry-with-a-self-signed-certificate-using-k) and [examples](https://github.com/coreos/tectonic-docs/blob/master/Documentation/admin/add-registry-cert.md) on the topic.

With Kubernetes 1.20+ and `containerd`, the principle is the same, but `containerd` has of course a different configuration system and the process is thus a little different. Also, at the time of writing, `containerd` needs to be restarted in order to pick up the new certificate, which means that there is a need for more than just copying files to the node.

Here is below a stab at solving this problem using a new DaemonSet spec: 

- It uses a privileged security context and `hostPID` in order to be able to execute commands on the nodes.
- The privileged commands are run in an `initContainers`, then an un-privileged container is left running.
- The commands to run are just a one-liner, stored in a ConfigMap so they can be modified independantly of the DaemonSet. 
- It uses a vanilla base image, Debian here, but it should work with any image that has the `nsenter` tool installed (also tested with Alpine). No need for a custom image.
- No attempt is made to modify the `containerd` configuration: I am not sure it is wise to modify these config files on the host, especially if you are using a hosted Kubernetes cluster, like Azure Kubernetes Service (AKS). Instead, the certificate is installed in the system location for trusted certificates.
- Finally, `containerd` is restarted which seems unavoidable currently.

I hope this can be useful to you, and don't forget to check for [newer versions of `containerd`](https://github.com/containerd/containerd) that could offer better ways of performing this configuration change!

{% gist 25f7db5b419c24db8bc6cac2fa4c2a13 containerd-certificate-ds.yaml %}
