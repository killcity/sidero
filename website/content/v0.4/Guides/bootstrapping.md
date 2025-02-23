---
description: "A guide for bootstrapping Sidero management plane"
weight: 1
title: "Bootstrapping"
---

## Introduction

Imagine a scenario in which you have shown up to a datacenter with only a laptop and your task is to transition a rack of bare metal machines into an HA management plane and multiple Kubernetes clusters created by that management plane.
In this guide, we will go through how to create a bootstrap cluster using a Docker-based Talos cluster, provision the management plane, and pivot over to it.
Guides around post-pivoting setup and subsequent cluster creation should also be found in the "Guides" section of the sidebar.

Because of the design of Cluster API, there is inherently a "chicken and egg" problem with needing a Kubernetes cluster in order to provision the management plane.
Talos Systems and the Cluster API community have created tools to help make this transition easier.

## Prerequisites

First, you need to install the latest `talosctl` by running the following script:

```bash
curl -Lo /usr/local/bin/talosctl https://github.com/talos-systems/talos/releases/latest/download/talosctl-$(uname -s | tr "[:upper:]" "[:lower:]")-amd64
chmod +x /usr/local/bin/talosctl
```

You can read more about Talos and `talosctl` at [talos.dev](https://www.talos.dev/latest).

Next, there are two big prerequisites involved with bootstrapping Sidero: routing and DHCP setup.

From the routing side, the laptop from which you are bootstrapping _must_ be accessible by the bare metal machines that we will be booting.
In the datacenter scenario described above, the easiest way to achieve this is probably to hook the laptop onto the server rack's subnet by plugging it into the top-of-rack switch.
This is needed for TFTP, PXE booting, and for the ability to register machines with the bootstrap plane.

DHCP configuration is needed to tell the metal servers what their "next server" is when PXE booting.
The configuration of this is different for each environment and each DHCP server, thus it's impossible to give an easy guide.
However, here is an example of the configuration for an Ubiquti EdgeRouter that uses vyatta-dhcpd as the DHCP service:

This block shows the subnet setup, as well as the extra "subnet-parameters" that tell the DHCP server to include the ipxe-metal.conf file.

> These commands are run under the `configure` option in EdgeRouter

```bash
$ show service dhcp-server shared-network-name MetalDHCP

 authoritative enable
 subnet 192.168.254.0/24 {
     default-router 192.168.254.1
     dns-server 192.168.1.200
     lease 86400
     start 192.168.254.2 {
         stop 192.168.254.252
     }
     subnet-parameters "include &quot;/etc/dhcp/ipxe-metal.conf&quot;;"
 }
```

Here is the `ipxe-metal.conf` file.

```bash
$ cat /etc/dhcp/ipxe-metal.conf

allow bootp;
allow booting;

next-server 192.168.1.150;
filename "ipxe.efi"; # use "undionly.kpxe" for BIOS netboot or "ipxe.efi" for UEFI netboot

host talos-mgmt-0 {
    fixed-address 192.168.254.2;
    hardware ethernet d0:50:99:d3:33:60;
}
```

> If you want to boot multiple architectures, you can use the *DHCP Option 93* to specify the architecture.

First we need to define *option 93* in the DHCP server configuration.

```bash
set service dhcp-server global-parameters "option system-arch code 93 = unsigned integer 16;"
```

Now we can specify condition based on *option 93* in `ipxe-metal.conf` file

```bash
$ cat /etc/dhcp/ipxe-metal.conf

allow bootp;
allow booting;

next-server 192.168.1.150;

if option system-arch = 00:0b {
    filename "ipxe-arm64.efi";
} else {
    filename "ipxe.efi";
}

host talos-mgmt-0 {
    fixed-address 192.168.254.2;
    hardware ethernet d0:50:99:d3:33:60;
}
```

Notice that it sets a static address for the management node that I'll be booting, in addition to providing the "next server" info.
This "next server" IP address will match references to `PUBLIC_IP` found below in this guide.

## Create a Local Cluster

The `talosctl` CLI tool has built-in support for spinning up Talos in docker containers.
Let's use this to our advantage as an easy Kubernetes cluster to start from.

Set an environment variable called `PUBLIC_IP` which is the "public" IP of your machine.
Note that "public" is a bit of a misnomer.
We're really looking for the IP of your machine, not the IP of the node on the docker bridge (ex: `192.168.1.150`).

```bash
export PUBLIC_IP="192.168.1.150"
```

We can now create our Docker cluster.
Issue the following to create a single-node cluster:

```bash
talosctl cluster create \
 --kubernetes-version 1.22.2 \
  -p 69:69/udp,8081:8081/tcp \
  --workers 0 \
  --endpoint $PUBLIC_IP
```

Note that there are several ports mentioned in the command above.
These allow us to access the services that will get deployed on this node.

Once the cluster create command is complete, issue `talosctl kubeconfig /desired/path` to fetch the kubeconfig for this cluster.
You should then set your `KUBECONFIG` environment variable to the path of this file.

## Untaint Control Plane

Because this is a single node cluster, we need to remove the "NoSchedule" taint on the node to make sure non-controlplane components can be scheduled.

```bash
kubectl taint node talos-default-master-1 node-role.kubernetes.io/master:NoSchedule-
```

## Install Sidero

As of Cluster API version 0.3.9, Sidero is included as a default infrastructure provider in clusterctl.

To install Sidero and the other Talos providers, simply issue:

```bash
SIDERO_CONTROLLER_MANAGER_HOST_NETWORK=true \
  SIDERO_CONTROLLER_MANAGER_API_ENDPOINT=$PUBLIC_IP \
  clusterctl init -b talos -c talos -i sidero
```

We will now want to ensure that the Sidero services that got created are publicly accessible across our subnet.
These variables above will allow the metal machines to speak to these services later.

## Register the Servers

At this point, any servers on the same network as Sidero should PXE boot using the Sidero PXE service.
To register a server with Sidero, simply turn it on and Sidero will do the rest.
Once the registration is complete, you should see the servers registered with `kubectl get servers`:

```bash
$ kubectl get servers -o wide
NAME                                   HOSTNAME        ACCEPTED   ALLOCATED   CLEAN
00000000-0000-0000-0000-d05099d33360   192.168.254.2   false      false       false
```

## Setting up IPMI

Sidero can use IPMI information to control Server power state, reboot servers and set boot order.
IPMI information will be, by default, setup automatically if possible as part of the acceptance process.
See [IPMI](../../resource-configuration/servers/#ipmi) for more information.

IPMI connection information can also be set manually in the Server spec after initial registration:

```bash
kubectl patch server 00000000-0000-0000-0000-d05099d33360 --type='json' -p='[{"op": "add", "path": "/spec/bmc", "value": {"endpoint": "192.168.88.9", "user": "ADMIN", "pass":"ADMIN"}}]'
```

If IPMI info is not set, servers should be configured to boot first from network, then from disk.

## Configuring the installation disk

Note that for bare-metal setup, you would need to specify an installation disk.
See [Installation Disk](../../resource-configuration/servers/#installation-disk) for details on how to do this.
You should configure this before accepting the server.

## Accept the Servers

Note in the output above that the newly registered servers are not `accepted`.
In order for a server to be eligible for consideration, it _must_ be marked as `accepted`.
Before a `Server` is accepted, no write action will be performed against it.
Servers can be accepted by issuing a patch command like:

```bash
kubectl patch server 00000000-0000-0000-0000-d05099d33360 --type='json' -p='[{"op": "replace", "path": "/spec/accepted", "value": true}]'
```

For more information on server acceptance, see the [server docs](../../resource-configuration/servers).

## Create Management Plane

We are now ready to template out our management plane.
Using clusterctl, we can create a cluster manifest with:

```bash
clusterctl generate cluster management-plane -i sidero > management-plane.yaml
```

Note that there are several variables that should be set in order for the templating to work properly:

- `CONTROL_PLANE_ENDPOINT` and `CONTROL_PLANE_PORT`: The endpoint (IP address or hostname) and the port used for the Kubernetes API server
  (e.g. for `https://1.2.3.4:6443`: `CONTROL_PLANE_ENDPOINT=1.2.3.4` and `CONTROL_PLANE_PORT=6443`).
  This is the equivalent of the `endpoint` you would specify in `talosctl gen config`.
  There are a variety of ways to configure a control plane endpoint.
  Some common ways for an HA setup are to use DNS, a load balancer, or BGP.
  A simpler method is to use the IP of a single node.
  This has the disadvantage of being a single point of failure, but it can be a simple way to get running.
- `CONTROL_PLANE_SERVERCLASS`: The server class to use for control plane nodes.
- `WORKER_SERVERCLASS`: The server class to use for worker nodes.
- `KUBERNETES_VERSION`: The version of Kubernetes to deploy (e.g. `v1.22.2`).
- `CONTROL_PLANE_PORT`: The port used for the Kubernetes API server (port 6443)
- `TALOS_VERSION`: This should correspond to the minor version of Talos that you will be deploying (e.g. `v0.13`).
  This value is used in determining the fields present in the machine configuration that gets generated for Talos nodes.

For instance:

```bash
export CONTROL_PLANE_SERVERCLASS=any
export WORKER_SERVERCLASS=any
export TALOS_VERSION=v0.13
export KUBERNETES_VERSION=v1.22.2
export CONTROL_PLANE_PORT=6443
export CONTROL_PLANE_ENDPOINT=1.2.3.4
clusterctl generate cluster management-plane -i sidero > management-plane.yaml
```

In addition, you can specify the replicas for control-plane & worker nodes in management-plane.yaml manifest for TalosControlPlane and MachineDeployment objects.
Also, they can be scaled if needed (after applying the `management-plane.yaml` manifest):

```bash
kubectl get taloscontrolplane
kubectl get machinedeployment
kubectl scale taloscontrolplane management-plane-cp --replicas=3
```

Now that we have the manifest, we can simply apply it:

```bash
kubectl apply -f management-plane.yaml
```

**NOTE: The templated manifest above is meant to act as a starting point.**
**If customizations are needed to ensure proper setup of your Talos cluster, they should be added before applying.**

Once the management plane is setup, you can fetch the talosconfig by using the cluster label.
Be sure to update the cluster name and issue the following command:

```bash
kubectl get talosconfig \
  -l cluster.x-k8s.io/cluster-name=<CLUSTER NAME> \
  -o yaml -o jsonpath='{.items[0].status.talosConfig}' > management-plane-talosconfig.yaml
```

With the talosconfig in hand, the management plane's kubeconfig can be fetched with `talosctl --talosconfig management-plane-talosconfig.yaml kubeconfig`

## Pivoting

Once we have the kubeconfig for the management cluster, we now have the ability to pivot the cluster from our bootstrap.
Using clusterctl, issue:

```bash
clusterctl init --kubeconfig=/path/to/management-plane/kubeconfig -i sidero -b talos -c talos
```

Followed by:

```bash
clusterctl move --to-kubeconfig=/path/to/management-plane/kubeconfig
```

Upon completion of this command, we can now tear down our bootstrap cluster with `talosctl cluster destroy` and begin using our management plane as our point of creation for all future clusters!
