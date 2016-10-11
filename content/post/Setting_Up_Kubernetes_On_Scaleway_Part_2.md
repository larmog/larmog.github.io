+++
date = "2016-10-01T11:00:00+02:00"
draft = true
title = "Setting up Kubernetes on Scaleway - Part 2"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "fabric8"]
categories = [ "Development", "Kubernetes", "Docker", "Cloud" ]
+++

<a href="http://scaleway.com">
<img src="https://www.scaleway.com/img/scaleway.f0e4.svg" style="float:right;height:80px"></a>

This is the second post in a series on setting up Kubernetes using the
bare-metal cloud provider Scaleway.
In [Part 1]({{< relref "Setting_Up_Kubernetes_On_Scaleway_Part_1.md" >}}) we
talked about setting up a master (controller) and a worker node.

This second post was supposed to be about securing your `api-server` and setting
up an `ingress-controller`. But I wasn't satisfied with how the network was
setup. The reason is that Scaleway doesn't support multi-tenant
network isolation or `VLAN`, so I wanted to solve that first. And so I did,
kind of, but in the mean time Kubernetes `v1.4.0` was released with a new tool
`kubeadm`.
<!--more-->

The tool `kubeadm` let´s you easily install a secure Kubernetes cluster. It
makes projects like my own `Sloop`, unnecessary.

So in this post we will take a step back and re-install Kubernetes and
setup an `ingress-controller`.

#### Prerequisites

First you need to create a Scaleway account and install the
[`scaleway-cli`](https://github.com/scaleway/scaleway-cli) tool.









A _Scaleway_ server always has a private (no public) ip and optionally a public
one. Public IP:s connects your servers to the Internet, private IP:s don't.
What that means is that servers without a public IP don't have access to the
Internet. We would prefer to let our nodes have only private IP:s making them
somewhat shielded from the Internet, but that stops us from fetching os updates
and docker images.

Another problem is, that even if we use a private IP our network is
shared with all other Scaleway users, located in the same data center.

The solution to these problems is to use a gateway with a public IP and let
our workers use it as the default gateway. The solution to the other problem and
make our network private is to use
[Weave Net](https://www.weave.works/products/weave-net/) for our __private__
network.

This post will show you how to install `weave` using `sloop` and setup a gateway
and configure it as the `default gateway` on all your `workers` and start
Kubernetes and verify the installation. So let's get started.

My _Scaleway Developer_ account only allows for creating two `C1` servers. And I
want to reserve these two servers for our `Kubernetes-On-ARM` cluster. So we
will setup three severs one `VC1S`(`x86-64`) used as gateway and two `C1`(`ARM`)
servers.

#### Prerequisites

First you need to create a Scaleway account and install the
[`scaleway-cli`](https://github.com/scaleway/scaleway-cli) tool.

#### Step 1. Setting up our gateway

First step is to create our `gateway`, lets call it `scw-gw`.

##### Create the server

```shell
$ # 1. Create your server
$ scw start -w $(scw create --name scw-gw --commercial-type=VC1S  Docker)
$
$ # 2. Find the public IP of your new server
$ scw ps
SERVER ID IMAGE          COMMAND  CREATED         STATUS   PORTS            NAME    COMMERCIAL TYPE
7f241abf  Docker_1_12_1           About a minute  running  xxx.xxx.xxx.xxx  scw-gw  VC1S
$
$ # 3. Login and set a passwd
$ ssh root@xxx.xxx.xxx.xxx
root@scw-gw:~# passwd
root@scw-gw:~# exit
$
$ # 4. You can now use scw to connect
$ scw attach scw-gw
```

##### Install Sloop

First install sloop:

```shell
root@scw-gw:~# curl -L https://raw.githubusercontent.com/kodbasen/sloop/develop/sloop -o /usr/local/bin/sloop; chmod +x /usr/local/bin/sloop
root@scw-gw:~# chmod +x /usr/local/bin/sloop
```

Next step is to start weave (using `sloop`).
```shell
root@scw-gw:~# sloop start weave
```

You can verify that `weave` is running using the command `weave status`.

##### Setup ip_forward

First we install some tools that let us persist out iptables rules:

```shell
root@scw-gw:~# apt-get update && apt-get -y install iptables-persistent
```

Next we list our network interfaces:

```shell
root@scw-gw:~# ip -4 addr show scope global
```
You should see three devices: `eth0`, `docker0` and `weave`. We wan't to enable
ip forwarding from `weave` to `eth0`. Next step is to enable `ip_forward`:

```shell
root@scw-gw:~# echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
root@scw-gw:~# # Edit /etc/sysctl.conf so that ip_forward is enabled after reboot
root@scw-gw:~# # # Uncomment net.ipv4.ip_forward=1
root@scw-gw:~# vi /etc/sysctl.conf
```

Next step is to add two rules to iptables and save them:

```shell
root@scw-gw:~# iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
root@scw-gw:~# iptables -A FORWARD -i weave -j ACCEPT
root@scw-gw:~# iptables-save
```

#### Step 2. Prepare our Kubernetes-On-ARM nodes

Next step is to create and setup our Kubernetes nodes. We will use `C1` `ARM`
servers.

##### Create the servers

Sorry to say, I haven't found any way of connecting to the servers without
setting a `root` password, and to be able to set a password you need to use `ssh`,
and to be able to use `ssh` you need a public IP address.

```shell
$ for i in {1..2}; do
>   scw start -w $(scw create --name scw-node$i --commercial-type=C1  Docker)
> done
03478d5d...
0494a9b2...
$ scw ps
```

Follow the same procedure as above for setting a `root` password. We can now
remove the public IP-address from the server and use `scw` to connect:

```shell
$ scw ps
SERVER ID    IMAGE            STATUS     NAME         COMMERCIAL TYPE
4243cdd5     Docker_1_12_1    running    scw-node2    C1
8d5432e5     Docker_1_12_1    running    scw-gw       VC1S
03d58d5d     Docker_1_12_1    running    scw-node1    C1
$ scw attach 03d58d5d
You are connected, type 'Ctrl+q' to quit.

root@scw-node1:~#
```
##### Install weave on ARM

Weave is not officially supported on `ARM` (yet), but I've built version `1.6.1`
that you can use. To install it, use the same method as above using `sloop`.

```shell
$ scw attach scw-node1
WEAVE_PASSWORD=ch@ng3m3 weave launch --ipalloc-init=observer 10.3.135.205
root@scw-node1:~# sloop start weave
```
##### Setting up default gateway

We need to fix so that our worker nodes can have Internet access. Currently the
default gateway is configured by Scaleway. We need to change that so that
the default gateway is our `scw-gw` and that we use the `weave` interface.

First we need to find the IP address of the `default gateway`:

```shell
root@scw-node1:~# ip route show | grep default
default via 10.1.xxx.1 dev eth0
```

Add a route to `scw-gw` through the `default gateway`:

```shell
root@scw-node1:~# route add <scw-gw-private-ip> gw 10.1.xxx.1
```

Now remove that default gateway:

```shell
root@scw-node1:~# ip route del 0/0
```

Add `scw-gw` as `default gateway` but use the `ip` of the `weave` interface
on `scw-gw`:

```shell
root@scw-node1:~# route add default gw 10.32.0.1 weave
```

Now all traffic will go through the default gateway except the `weave-peer`
that will use the static route. We should also add a route to the other `node`
so that `weave` can create a `mesh`.

I've wrote an earlier post about [GlusterFS On Kubernetes-ARM]({{< relref "GlusterFS_On_Kubernetes-ARM.md" >}}). GlusterFS will probably suit us well and I had hoped that this series would be all about setting up Kubernetes on `ARM`. But my _Scaleway Developer_ account only allows for creating two `C1` servers and I've filled my quota of `C1`:s. Instead we will be using `VC1S` but the process is the same.

If you haven't installed the [Scaleway CLI](https://github.com/scaleway/scaleway-cli) yet then it´s time to do so.

#### Setting up two glusterfs nodes

 Create two new servers for running `glusterfs-server`.  

```shell
for i in {1..2}; do
   scw start $(scw create --name scw-gfs$i  Ubuntu_Xenial_16_04)
done
```

Install `glusterfs-server` on the two new servers.

```shell
for i in {1..2}; do
   scw exec scw-gfs$i 'apt-get update && apt-get install -q -y glusterfs-server' < /dev/null &
done
```
