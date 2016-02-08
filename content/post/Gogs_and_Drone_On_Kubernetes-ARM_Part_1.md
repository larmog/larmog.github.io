+++
date = "2016-02-07T11:17:02+01:00"
draft = false
title = "Gogs and Drone On Kubernetes-ARM - Part 1"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "Gogs", "Drone"]
categories = [ "Development", "Kubernetes", "Docker" ]
+++

This is part 1 in a series of posts describing how I have setup Gogs and Drone
on my Kubernetes-ARM cluster. [Gogs - Go Git Service](https://gogs.io/) is
_A painless self-hosted Git service_ and is a great alternative when you can't
use GitHub or wan't to host your own Git service.
<!--more-->

The easiest way to get started with Gogs (and of course the only alternative if
you wan't to use Kubernetes) is to use a Docker image. Gogs has a Docker image
ready on Docker Hub. Unfortunately that image won't work on ARM.
Thankfully the Hypriot team has two Gogs Docker images ready: [hypriot/rpi-gogs-raspbian](https://hub.docker.com/r/hypriot/rpi-gogs-raspbian/)
and [hypriot/rpi-gogs-alpine](https://hub.docker.com/r/hypriot/rpi-gogs-alpine/)

Lets try it out:
```shell
$ kubectl run my-gogs --image=hypriot/rpi-gogs-alpine --replicas=1 --port=3000
replicationcontroller "my-gogs-service" created
```
Here we can see our pod is running:
```shell
$ kubectl get pods
NAME                      READY     STATUS    RESTARTS   AGE
my-gogs-3adh8             1/1       Running   0          1m
```

Now it's time to expose our new pod as a service:
```shell
$ kubectl expose rc my-gogs --port=80 --target-port=3000 --name=my-gogs-service
service "my-gogs-service" exposed
```

If all went well we can now access or new service:
```shell
$ curl -s http://[master-ip]:8080/api/v1/proxy/namespaces/default/services/my-gogs-service/install|grep Version:
<p class="left" id="footer-rights">© 2015 Gogs · Version: 0.6.1.0325 Beta Page:<strong>2259ms</strong>
```
But oops! that´s kind of an old version. Our goal is to use Gogs together with
Drone and this won't work. We need a version greater than `0.6.16.1022`
([see](http://readme.drone.io/setup/gogs/)). I guess this is the difference
between _leading edge_ and _bleeding edge_. Again we're saved by some one else's
work. Gogs has a `Dockerfile.rpi` ready that we can use to build our own image.
I've built and pushed an image to Docker Hub that you can use:
[`larmog/rpi-gogs`](https://hub.docker.com/r/larmog/rpi-gogs/) that is `33MB`.

So lets repeat the steps above with the new image and `curl` for the version:
```shell
$ curl -s http://[master-ip]:8080/api/v1/proxy/namespaces/default/services/my-gogs-service/install|grep Version
© 2016 Gogs Version: 0.8.23.0126 Page: <strong>1622ms</strong> Template: <strong>1619ms</strong>
```
Version `0.8.23.0126`, that looks so much better don't, you think?

Next step is to install Gogs. But hey... wait a minute - what about persistence?
We need to add a _Volume_. I'm using my home NAS, a DiskStation, over NFS. The
only thing we need to do is to share a volume over NFS and install NFS on our
nodes.

```shell
$ sudo apt-get -y install nfs-common
```

Next we need to create a `PersistentVolume`:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gogs
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /volume1/kbn1/gogs
    server: my-nfs-server
```
and then a `PersistentVolumeClaim`:
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-gogs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```
and lastly mount the volume in our pod template:
```
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    app: my-gogs
  name: my-gogs
  namespace: default
spec:
  replicas: 1
  selector:
    app: my-gogs
  template:
    metadata:
      labels:
        app: my-gogs
    spec:
      containers:
      - image: larmog/rpi-gogs:0.8.23.0126-2
        imagePullPolicy: IfNotPresent
        name: my-gogs
        volumeMounts:
        - mountPath: "/data"
          name: persistentdata
        resources: {}
        ports:
          - containerPort: 3000
            name: web   
            protocol: TCP
          - containerPort: 22
            name: ssh
            protocol: TCP
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
        - name: persistentdata
          persistentVolumeClaim:
            claimName: pvc-gogs
```

NFS v4 is kind of hard to use if you don't have synchronized your users and
groups in your domain. I use `all_squash` to a specific UID/GID in order to get
it to work with my NAS, and that works fine for Gogs but I've got plans to
replace NFS with [GlusterFS](https://www.gluster.org/) and it's on the `TODO`
list.

Let's enjoy the fruit of our work (or as we say in Sweden: "ett Ernst ögonblick"):
```shell
$ kubectl get services
NAME             CLUSTER_IP   EXTERNAL_IP   PORT(S)    SELECTOR       AGE
my-gogs-service  10.0.0.85    <none>        80/TCP     app=gogs       9d
my-gogs-ssh      10.0.0.216   nodes         2222/TCP   app=gogs       9d
kubernetes       10.0.0.1     <none>        443/TCP    <none>         25d
```
Notice that I've also created a service for the `ssh` port.
Now we can complete the Gogs installation. Open the url (http://[master-ip]:8080/api/v1/proxy/namespaces/default/services/my-gogs-service)
and complete the installation.

In [Part 2]({{< relref "Gogs_and_Drone_On_Kubernetes-ARM_Part_2.md" >}}) I will
explain how to set up `service-loadbalancer` to expose your services outside your
cluster.
