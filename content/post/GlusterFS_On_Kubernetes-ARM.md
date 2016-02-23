+++
date = "2016-02-22T09:35:30+01:00"
draft = false
title = "GlusterFS On Kubernetes ARM"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "GlusterFS", "Gogs", "Drone", "MySQL", "ARM"]
categories = [ "Development", "Kubernetes", "Docker" ]
+++

If you followed my earlier posts, you know that I’m running a Kubernetes cluster
on Raspberry Pi, using __HypriotOS__ and Lucas Käldströms [Kubernetes-On-ARM](https://github.com/luxas/kubernetes-on-arm) project.
<!--more-->
If not you can find my earlier posts here:

* [Kubernetes-On-ARM]({{< relref "2016-02-05-Kubernetes-On-ARM.md" >}})
* [Gogs and Drone On Kubernetes-ARM - Part 1]({{< relref "Gogs_and_Drone_On_Kubernetes-ARM_Part_1.md" >}})
* [Gogs and Drone On Kubernetes-ARM - Part 2]({{< relref "Gogs_and_Drone_On_Kubernetes-ARM_Part_2.md" >}})
* [Gogs and Drone On Kubernetes-ARM - Part 3]({{< relref "Gogs_and_Drone_On_Kubernetes-ARM_Part_3.md" >}})


I've used __NFS__ volumes for storage and DiskStation NAS as NFS server.
The solution have had some drawbacks and it's been hard to synchronize users and
groups, and I've used `all_squash` to a specific `uid`. That worked for __Gogs__
and __Drone__ but not for __MySQL__.

Another alternative is to use __GlusterFS__ volumes. GlusterFS is a clustered
file-system that is capable of scaling to several peta-bytes.
GlusterFS aggregates storage bricks, that can be made of commodity hardware.
When I started this project, I had two
`Raspberry Pi 1 B+` in my desk drawer. I've used them as nodes in the cluster,
but the small amount of `RAM` has shown them less useful compared to the newer
`Raspberry Pi 2 B` based nodes. But then it hit me that maybe I could use the
old Pi 1 boards as bricks in GlusterFS. One of the purposes with this project is
to see how much of a Kubernetes cluster you can get for a reasonable amount. So
I'd like to keep a tight budget. Normally when setting up a brick you want to
use RAID, because you don't want to handle disk failures on a brick level, but
for this project it's a reasonable solution.

After a trip to my local Electronic Retailers I came home with this
(not the cluster):
{{< img src="/media/IMG_1953.jpg" title="GlusterFS ingredients" >}}

* * 2 WD external usb harddrives
* * 2 Raspberry Pi 2 B boards (to replace the Pi 1:s)
* * 2 SD cards
* * One more Multi-Pi Stackable Raspberry Pi Case
* * Cables

My first attempt was to connect the external drives to the usb port. That didn't
work of course, external disks needs more power than what the Pi usb port can
deliver. There are two alternatives for solving this. You could use an external
usb-hub with it's own power supply. But that means two usb-hubs and two power
adapters. The other alternative is to use a usb Y-cable and connect the extra
connector to the usb supercharger (that powers all nodes). After one more trip
to the store to get a Y-cable and then connect it to the usb supercharger, the
problem was solved.

The image shows the cluster after assembling.
{{< img src="/media/IMG_1961.jpg" title="Cluster ready for GlusterFS" >}}

There's room for two more Pi:s in the switch and usb supercharger.

Next step was to partition and format the disks:

```shell
$ # Prepare your disk and create a partition first
$ # Install xfs tools
$ apt-get install -y xfsprogs
$ # Format the disk
$ mkfs.xfs -f -L brick1 -i size=512 /dev/sda1
```
... and mount and install `glusterfs`.
```shell
$ # Create and mount the brick
$ mkdir -p /data/brick1
$ echo '/dev/sda1 /data/brick1 xfs defaults 1 2' >> /etc/fstab
$ cat /etc/mtab
# Install GlusterFS server
$ apt-get install -y glusterfs-server
```
The steps above was made on both servers.

Next we needed to connect our storage servers:
```shell
$ #From server1
$ gluster peer probe server2
```
and:
```shell
$ #From server2
$ gluster peer probe server1
```

Time to create the first volume:
```shell
$ mkdir /data/brick1/vol0
$ gluster volume create vol0 replica 2 store1.local:/data/brick1/vol0 store2.local:/data/brick2/vol0
$ gluster volume start vol0
$ gluster volume info
$ gluster volume info

Volume Name: vol0
Type: Replicate
Volume ID: 6ffe7723-e0ac-44c7-a004-fa951467884d
Status: Started
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: server1:/data/brick1/vol0
Brick2: server2:/data/brick2/vol0
```
And we're done.

The only thing left is to set up our `glusterfs-cluster endpoints` as described
in the [Kubernetes documentation for GlusterFS Volumes](http://kubernetes.io/v1.1/examples/glusterfs/README.html) and configure
our [Persistent Volumes and Claims](http://kubernetes.io/v1.1/docs/user-guide/persistent-volumes.html)

Migrating the volumes from NFS to GlusterFS was very easy. The only problem
I've had was a silly mistake. Coming from NFS I first tried to share a volume
between applications, using different paths: `/vol 0/mysql`, `/vol/drone` that
didn't work. You need to create different volumes for each application. Except
from the small misconception, there's been no problems at all.

Now finally I have a Kubernetes cluster running on bare metal that can be used
for trying out techniques, for building microservices.

By the way, I took the opportunity to overclock my Raspberry Pi:s.
Thanks to [Hayden James](http://haydenjames.io/raspberry-pi-2-overclock/), all
cluster nodes now runs at 1000 Mhz, and it seems stable.
