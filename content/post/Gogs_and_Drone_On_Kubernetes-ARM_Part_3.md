+++
date = "2016-02-08T16:39:34+01:00"
draft = false
title = "Gogs and Drone On Kubernetes-ARM - Part 3"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "Gogs", "Drone"]
categories = [ "Development", "Kubernetes", "Docker", "CI" ]

+++

This is the third part in a series on setting up Gogs and Drone on
Kubernetes-ARM. You can find the earlier posts here:

* [Kubernetes-On-ARM]({{< relref "2016-02-05-Kubernetes-On-ARM.md" >}})
* [Gogs and Drone ... Part 1]({{< relref "Gogs_and_Drone_On_Kubernetes-ARM_Part_1.md" >}})
* [Gogs and Drone ... Part 2]({{< relref "Gogs_and_Drone_On_Kubernetes-ARM_Part_2.md" >}})


In this third episode I will try to show how to setup Drone on Kubernetes-ARM.
A disclaimer, I'm new to Drone and out on deep water so...
<!--more-->

If your wondering about Drone take a look at
[GitHub page](https://github.com/drone/drone).

_"Every build is executed inside an ephemeral Docker container, giving
developers complete control over their build environment with guaranteed
isolation."_

This means that the `build agent` is a container and all the plugins are also
containers. This makes it a bit problematic running Drone on ARM because we
need to rebuild everything, from Drone it self to all plugins that we wan't to
use. We need to setup a Drone infrastructure for ARM, __or so I thought...__

##### On the shoulder of others
Fortunately before I jumped in to the _rabbit hole_, not knowing if it was a
dead end, I found two different initiatives for running Drone on ARM. You can
read more about it here:
[Drone ported to ARM](https://discuss.drone.io/t/drone-ported-to-arm/55)

I decided to try
[armhf-docker-library/drone](https://github.com/armhf-docker-library/drone) and
I also found this example from Greg Taylor: [drone-on-kubernetes](https://github.com/drone-demos/drone-on-kubernetes/).

##### Lets get started
First we need to create persistent volume for our sqlite database:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-drone
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: <Path exported NFS volume>
    server: <IP NFS Server>
```

and a volume claim:
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-drone
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

We can now create our replication controller:
```
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    app: droneio
  name: droneio
  namespace: default
spec:
  replicas: 1
  selector:
    app: droneio
  template:
    metadata:
      labels:
        app: droneio
    spec:
      containers:
      # Drone ARM port https://github.com/armhf-docker-library
      - image: armhfbuild/drone:latest
        imagePullPolicy: Always
        name: droneio
        ports:
          - containerPort: 8000
            name: web   
            protocol: TCP
        env:
        - name: REMOTE_DRIVER
          value: "gogs"
        - name: REMOTE_CONFIG
          value: "https://<YOUR GOGS SERVICE>?skip_verify=true&open=false"
        - name: DEBUG
          value: "true"
        #Drone ARM port plugins
        - name: PLUGIN_FILTER
          value: "armhfplugins/*"
        volumeMounts:
          - mountPath: "/var/lib/drone"
            name: persistentdata
          - mountPath: /var/run/docker.sock
            name: docker-socket
          - mountPath: /var/lib/docker
            name: docker-lib
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
          - name: persistentdata
            persistentVolumeClaim:
              claimName: pvc-drone
          - name: docker-socket
            hostPath:
              path: /var/run/docker.sock
          - name: docker-lib
            hostPath:
              path: /var/lib/docker
```

Let's take a look at our controllers:
```shell
$ kubectl get rc
CONTROLLER   CONTAINER(S)   IMAGE(S)                        SELECTOR       REPLICAS   AGE
droneio      droneio        armhfbuild/drone:latest         app=droneio    1          2h
gogs         gogs           larmog/rpi-gogs:0.8.23.0126-2   app=gogs       1          10d
```


Finally we create our service:
```
apiVersion: v1
kind: Service
metadata:
  annotations:
    # For our service-loadbalancer
    serviceloadbalancer/lb.sslTerm: "true"
    serviceloadbalancer/lb.host: <virtual host>
    serviceloadbalancer/lb.cookie-sticky-session: "true"
  labels:
    app: droneio
  name: droneio
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8000
  selector:
    app: droneio
  sessionAffinity: None
  type: ClusterIP
```

And here´s our services:
```shell
$ kubectl get svc
NAME         CLUSTER_IP   EXTERNAL_IP   PORT(S)    SELECTOR       AGE
droneio      10.0.0.11    <none>        80/TCP     app=droneio    2h
gogs         10.0.0.85    <none>        80/TCP     app=gogs       13d
gogs-ssh     10.0.0.216   nodes         2222/TCP   app=gogs       13d
kubernetes   10.0.0.1     <none>        443/TCP    <none>         29d
```

If all has gone according to our plan we should see the service in our
`ha-proxy` status page, and you should be able to login with your _Gogs_
account.

##### CA and PKI infrastructure
If your using your own CA and PKI infrastructure and a service-loadbalancer,
then there is a couple of things you need to do. When you have activated a
repository in Drone, you need to edit the Webhook that notifies Drone.
Use the service name or the IP address of your Drone service
(`droneio.<namespace>`) in Kubernetes.
{{< img src="/media/gogs-webhook.jpg" title="Gogs Webhook" >}}

There is no easy way (at least none that i know of) to add your CA certificate.
This means that the `plugins/drone-git` will fail to clone your repository using
`https`. It gave me a serious headache until i found a solution and I admit it's
a bit of a hack.
{{< img src="/media/drone-git-failed.jpg" title="drone-git failed to clone" >}}

You need to set `GIT_SSL_NO_VERIFY=true` in your `.drone.yml`:
```
clone:
    environment:
      - GIT_SSL_NO_VERIFY=true
    path: /xxxx
```
{{< img src="/media/drone-git-success.jpg" title="drone-git cloned successfully" >}}

##### Conclusion
This concludes our series about running Gogs and Drone on Kubernetes-ARM.
Thanks to the excellent job done by others, it is a rather straight forward
process to set it up. I hope you liked the series and hope you've picked up
something you didn't know before. Now I'm off to learn some more about _Drone_.

##### Finally
Some useful links:

* http://blog.hypriot.com/
* https://github.com/luxas/kubernetes-on-arm
* http://www.jinkit.com/k8s-on-rpi/
* https://discuss.drone.io/t/drone-ported-to-arm/55
* https://github.com/kelseyhightower/docker-kubernetes-tls-guide
* https://github.com/kelseyhightower/conf2kube
* https://github.com/drone-demos
