+++
date = "2017-03-07T11:08:59+01:00"
title = "Minio, simple storage for your cluster"
draft = false
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Minio", "Storage"]
categories = [ "Development", "Kubernetes", "Docker", "Cloud" ]
+++

<img src="/media/minio-k8s.png" style="float:right;height:150px">
I've been using different volume types for storage in my `kubernetes-on-arm`
cluster. I wrote two posts about it: [GlusterFS On Kubernetes ARM]({{< relref "GlusterFS_On_Kubernetes-ARM.md" >}}) and [Storage for your cluster]({{< relref "Storage_for_your_cluster.md" >}}) a year ago. All solutions adds complexity
to the cluster, the one that's been easiest to handle is `nfs`. But `nfs` isn't transparent enough. I always seems to end up in strange corner cases where it’s
not working as expected. The `emptyDir` volume type let's you mount an empty
directory from the host in to your Pod. But when the Pod goes away, so does your
volume, leaving you with nothing.
<!--more-->

There is now beta features in Kubernetes `v1.5` for `StatefulSet` and
`init-containers` that got me started thinking. If one could use something
like [Minio](https://minio.io/) for storage in combination with `emptyDir`
volumes, maybe we could end up with a *good enough* solution that lowers the
technology stack.

The plan was to have an `init-container` running `minio mc` to mirror
in files from `minio server` using the `mirror` command to an
`emptyDir` volume in a `pod` defined by a `StatefulSet`. Once the `pod` is up
and running, file changes needs to be mirrored back to the `minio server`, so
that we don't lose any data. The `minio server` is also a `pod` defined by a
`StatefulSet` but uses a `nfs persistentVolume` for storage.

{{< img src="/media/minio-k8s-storage.png" title="Minio for simple storage" >}}

The plan was to use the `mc mirror --watch` command for mirroring changes back
to the `minio server`, and that is probably the most suitable solution, but there
was a [bug](https://github.com/minio/mc/issues/2047) in the `arm64` version so I
had to create a work around.
The work around is a simple script that for every `#` seconds checks for changes
and copies them using the  `mc cp` command back to the server.

If you wan't to try it out your self there's *multiarch* images for `arm` and
`arm64` and a `mysql` example at [minio-k8s-storage](https://github.com/TheNatureOfSoftware/minio-k8s-storage).

#### Starting `minio server`

The `accessKey` and `secretKey` is only for demo, you should create your own
config files using `kubectl create secret generic` and mount it to your container at
`/app/config.json`.

```shell

$ # the minio server expects a persistentVolume of at least 5Gi
$ kubectl create -f https://raw.githubusercontent.com/TheNatureOfSoftware/minio-k8s-storage/master/minio.yml
service "minio" created
statefulset "minio" created
$
$ kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
minio-0   1/1       Running   0          22s

```

Once the `minio server` is started you can access the ui using `port-forward`:

```shell

$ # port-forward to your minio server and create a store0 bucket
$ kubectl port-forward minio-0 9000:9000
Forwarding from 127.0.0.1:9000 -> 9000
Forwarding from [::1]:9000 -> 9000

```

Now you can login to `minio server`: http://localhost:9090 and create a bucket
named `store0`.

#### Starting the `mysql` example

```shell

$ kubectl create -f https://raw.githubusercontent.com/TheNatureOfSoftware/minio-k8s-storage/master/mysql-example.yml
service "mysql" created
statefulset "mysql" created
$
$ kubectl get pods
NAME      READY     STATUS     RESTARTS   AGE
minio-0   1/1       Running    0          3m
mysql-0   0/2       Init:0/2   0          8s
$ # you can see above, the two init containers has not yet run
$
$ kubectl get pods
NAME      READY     STATUS            RESTARTS   AGE
minio-0   1/1       Running           0          4m
mysql-0   0/2       PodInitializing   0          28s
$ # here you can see the pod is initializing
$
$ kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
minio-0   1/1       Running   0          4m
mysql-0   2/2       Running   0          49s
$ # and finally our mysql database is up and running

```

If you go back to your `minio server`: http://localhost:9090 and take
a look at the `store0` bucket, you should see all `mysql` files mirrored back.

##### A closer look whats happening

Here you can se that our `mysql` example has two `init-containers`.
They are executed in order and the first (`setup`) mirrors all files
to the `persistentdata` volume. The second container restores the file
ownership.

```

pod.beta.kubernetes.io/init-containers: '[{
  "name": "setup",
  "image": "thenatureofsoftware/mc",
  "args": ["mirror", "minio/store0", "/var/lib/mysql"],
  "volumeMounts": [{
    "mountPath": "/var/lib/mysql",
    "name": "persistentdata"
  }]
},
{
  "name": "chown",
  "image": "tobi312/rpi-mysql:5.6",
  "command": ["chown", "-R", "mysql:mysql", "/var/lib/mysql"],
  "volumeMounts": [{
    "mountPath": "/var/lib/mysql",
    "name": "persistentdata"
  }]
}]'

```

The last pice of the puzzle is the `sidecar` container to `mysql` that mirrors
changes back:

```

- image: thenatureofsoftware/mc-mirror:latest
        imagePullPolicy: Always
        name: mc-mirror
        args:
        - /var/lib/mysql
        - minio/store0
        volumeMounts:
        - mountPath: "/var/lib/mysql"
          name: persistentdata
      restartPolicy: Always

```

#### To sum up

The solution has it's pros and cons. The pros are that it uses the nodes
local disk when using volume type `emptyDir`. Once the volume is setup then there
should be very few surprises. The problem is of course that you need to
mirror everything back and the size of the local disk.

There are room for improvements like using the [Downward API](https://kubernetes.io/docs/tasks/configure-pod-container/downward-api-volume-expose-pod-information/#the-downward-api) for creating buckets.

Mysql is probably one of the worst examples I've could have picket. I'll
try to add some more like `zookeeper`, `cassandra` and `elastic`.

#### Links

* [Minio Cloud Storage](https://minio.io/)
* [minio-k8s-storage](https://github.com/TheNatureOfSoftware/minio-k8s-storage)
* [StatefulSet](https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/)
* [Init Containers](https://kubernetes.io/docs/concepts/abstractions/init-containers/)
