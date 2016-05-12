+++
date = "2016-05-12T06:48:19+02:00"
draft = false
title = "Storage for your cluster"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "ARM", "NFS", "glusterFS", "iscsi"]
categories = [ "Development", "Kubernetes", "Docker", "Storage" ]
+++

I've been working on getting Elasticsearch working on Kubernetes-On-ARM. The
biggest problem has been storage. I'm using Elasticserach for storing logs and
the cluster generates 1.4 million entries per day (i know, need to do
something about it).
{{< img src="/media/es-hits.png" title="Hits during 24 hours" >}}
If you want to deploy your applications to Kubernetes, sooner or later you need
to think about how to solve storage and persistence.
<!--more-->

Kubernetes is a distributed cluster where nodes and pods comes and goes. We
don't want to solve our persistence problem for a single node, we want to solve
it for the whole cluster. That's where Kubernetes Volumes comes in. A quick look
at the different [Volume Plugins](http://kubernetes.io/docs/user-guide/volumes/)
that are available gives us the following alternatives:

* `nfs`
* `iscsi`
* `glusterfs`

I wrote an earlier post about
[GlusterFS On Kubernetes-ARM]({{< relref "GlusterFS_On_Kubernetes-ARM.md" >}}).
I still use `glustefs` and it's running on my `rpi-1`
boards, but the I/O performance isn't enough for handling the logs from
Fluentd that are stored in Elasticsearch. We need a bigger machinery.

I've been looking for a board that have Gigabit Ethernet and SATA or
USB 3, that can be used for handling persistence. But buying a board and some
disks and configuring services, that sounds a lot like building your own NAS,
and there are plenty of cheap NAS products out there that supports at least two
of the alternatives on our list, `nfs` and `iscsi`.

I already own a home `NAS` and it supports both `nfs` and `iscsi`.

I created two different `PersistentVolume`:s, one for each Volume Plugin, and
mounted them in two different Elasticsearch data node pods, and suddenly, all the
problems with Elasticsearch and Kibana is gone.

##### Using iSCSI

Installing `iSCSI` Initiator in Arch Linux is rather straight forward:
```shell
$ pacman -S --noconfirm open-iscsi
$ systemctl enable open-iscsi.service
$ systemctl restart open-iscsi.service
$ iscsiadm -m discovery -t sendtargets -p <ip of your iscsi target>
```
You need to repeat the steps above for each node in your cluster and that's it,
if your not going to use `CHAP`.

Now we can create a `PersistentVolume`:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: lun0
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  iscsi:
    targetPortal: xxx.xxx.xxx.xxx:yyyy
    iqn: <iqn-target>
    lun: 0
    fsType: ext4
    readOnly: false
```

I leave it up to you to create a `PersistentVolumeClaim` and mount it in to your
pod.

#### To sum up

If you want to use your Kubernetes-On-ARM cluster for more advanced applications
then you need to think about how to solve persistence.

I've found a solution that suits my needs using `iscsi` and `nfs`, the only
problem is that my NAS is almost full, and not so portable.

Maybe it's time to build
`Kodbasen cluster version 2`?
