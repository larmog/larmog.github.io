+++
date = "2016-10-15T20:58:26+02:00"
draft = false
title = "Kubernetes on Scaleway - Part 3"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "GlusterFS", "Scaleway"]
categories = [ "Development", "Kubernetes", "Docker", "Cloud" ]
+++

This is the final part in a series about setting up Kubernetes on *Scaleway*.
This final part is about setting up storage for your cluster.
<img src="/media/k8s-glusterfs-scaleway.png" style="float:right;height:150px">
Most of the steps here is already described in an earlier post:
[*GlusterFS On Kubernetes ARM*]({{< relref "GlusterFS_On_Kubernetes-ARM.md" >}})
 that I wrote a couple of month back. You can also find the earlier posts in this
series here:
[*Part 1*]({{< relref "Setting_Up_Kubernetes_On_Scaleway_Part_1.md" >}})
and [*Part 1 (revisited) and Part 2*]({{< relref "Kubernetes_on_Scaleway_Part_1_and_2.md" >}}).

<!--more-->
<p style="margin-bottom:40px"></p>

The solution described here is not the only way you can use to set up *GlusterFS*
for your cluster. You can also use a `DaemonSet` or a `PetSet` and run your
`glusterfs-servers` as containers, but I like the separation of concerns, using
two dedicated servers.

#### Setting up `glusterfs-server` nodes

I've chosen to use two servers for a replica set of 2. The first thing we need
to do is to create two additional servers.

```shell
$ for i in {1..2}; do
    scw start $(scw create --name gfs-$i --commercial-type="VC1S" Ubuntu_Xenial)
done
```
... and install `glusterfs-server` and connect the servers using their private
DNS name:

```shell
$ apt-get update && apt-get install -y glusterfs-server attr
$ gluster peer probe xxxxxx.priv.cloud.scaleway.com
```
##### Creating volumes

I create the volumes under the root partition, which is not recommended.
I leave it up to you mount an additional volume (disk) on your nodes for
`glusterfs volume` storage. We will create four volumes.

Create the volume directories on node `gfs-1`:

```shell
$ for i in {0..3}; do
    mkdir -p /data/brick2/vol$i
done
```
... and on `gfs-2`, then create the volumes:

```shell
$ # Create and start the volumes
$ for i in {0..3}; do
    mkdir -p /data/brick2/vol$i \
    && gluster volume create vol$i replica 2 \
    <your gfs-1 machine id>.priv.cloud.scaleway.com:/data/brick1/vol$i \
    <your gfs-2 machine id>.priv.cloud.scaleway.com:/data/brick2/vol$i force \
    && gluster volume start vol$i
done
$ # Check your volumes
$ gluster volume info
```
The `force` flag is used because we're using the root partition, and should not
be required if you use another partition.

#### Install `glusterfs-client` on your k8s nodes

Next step is to install `glusterfs-client` on all your *Kubernetes* nodes:

```shell
$ apt-get install -y glusterfs-client attr
```

#### Create the `glusterfs-endpoint`

The final step to setup *GlusterFS* is to create the endpoint.

Use `kubectl` and create the following `endpoint` and `service`, don't
forget to replace the IP:s with your own privare IP:s of your `gfs-1` and
`gfs-2` servers:

```
apiVersion: v1
kind: Endpoints
metadata:
  name: glusterfs-cluster
  namespace: default
subsets:
- addresses:
  - ip: 10.x.x.x
  - ip: 10.x.x.y
  ports:
  - port: 1
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: glusterfs-cluster
  namespace: default
spec:
  ports:
  - port: 1
    protocol: TCP
    targetPort: 1
```

And finally we can create four `PersistentVolume`:s using our *GlusterFS*
volumes.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol0
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  glusterfs:
    endpoints: glusterfs-cluster
    path: vol0
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  glusterfs:
    endpoints: glusterfs-cluster
    path: vol1
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol2
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  glusterfs:
    endpoints: glusterfs-cluster
    path: vol2
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol3
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  glusterfs:
    endpoints: glusterfs-cluster
    path: vol3
    readOnly: false
```

You can also create `PersistentVolumeClaim`:s that uses your `PersistentVolume`:s
but maybe it's better to wait until it's time to use them from your apps.
Notice that I have used `ReadWriteOnce` on all nodes and
`persistentVolumeReclaimPolicy: Retain`. This means that the volumes can not be
shared and you need to reclaim them manually.

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-0
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-2
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-3
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

#### A final word about file permissions

Depending on if your containers is running as root (`UID 0`) or any other `UID`
you may need to configure your `securityContext`.

#### Wrap up

This was the final part on setting up *Kubernetes* on *Scaleway*. The
installation can obviously be improved, for example the `traefik` proxy
does currently not support `url re-writes` which may be a problem for you. But
you can always replace it with `nginx`. To protect your `glusterfs-server` nodes
and save some IP-adresses, you can remove the public IP:s from your servers.
But remember you need to assign one, if you want to upgrade the server.

As always, feedback is very welcome.
