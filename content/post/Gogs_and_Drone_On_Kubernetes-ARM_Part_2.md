+++
date = "2016-02-08T09:21:47+01:00"
draft = true
title = "Gogs and Drone On Kubernetes-ARM - Part 2"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "Gogs", "Drone"]
categories = [ "Development", "Kubernetes", "Docker" ]
+++

This is the second part in a series of posts describing how I have setup Gogs
and Drone on
[Kubernetes-On-ARM]({{< relref "2016-02-05-Kubernetes-On-ARM.md" >}}) cluster.
In [Part 1]({{< relref "Gogs_and_Drone_On_Kubernetes-ARM_Part_1.md" >}}) we
talked about setting up Gogs.

In this part I'll explain how to setup `service-loadbalancer` to expose services
outside the cluster.
<!--more-->

I'm using [Lucas Käldström](https://github.com/luxas):s great
[kubernetes-on-arm](https://github.com/luxas/kubernetes-on-arm) project.
There's a load-balancer addon based on [kubernetes/contrib/service-loadbalancer](https://github.com/kubernetes/contrib)
that wasn't ready in the `0.6.3` release.

Before we begin you might take a look at
[service-loadbalancer](https://github.com/kubernetes/contrib/tree/master/service-loadbalancer).

What I did was to build the `service-loadbalancer` from the
`kubernetes/contrib master branch`. Once again Lucas have made a great job and
created a docker file for building the Kubernetes-ARM binaries on x86.

```shell
$ # Clone the repository
$ git clone https://github.com/luxas/kubernetes-on-arm
$ # Build the Docker image
$ cd kubernetes-on-arm/scripts/build-k8s-on-amd64
$ docker build -t build-k8s-on-amd64 .
$ # Create a container
$ docker run --name=build-k8s-on-amd64 build-k8s-on-amd64 true
$ # Copy out the binaries from the container
$ docker cp build-k8s-on-amd64:/output .
```
The `service-loadbalancer` binary is located in the `output` directory.
Transfer the file to the nodes that will have the role of load-balancer.
e.g.
```shell
$ scp ./output/service-loadbalancer \ xxx.xxx.xxx.xxx:/etc/kubernetes/source/images/kubernetesonarm/_bin/latest
```
The `service-loadbalancer`uses a template for creating a `ha-proxy.cfg`.
The template I'm using can be found here: [template.cfg](https://goo.gl/TzvKhX)
On each node build the `kubernetesonarm/loadbalancer` image.
```shell
$ cd /etc/kubernetes/source/images/kubernetesonarm/loadbalancer
$ mv template.cfg template.cfg.org
$ wget wget https://goo.gl/TzvKhX -O template.cfg
$ ./build.sh
```

Phew... were half way. The Docker image is in place on our nodes. Now we need
to label them so that the scheduler can place the `service-loadbalancer` on the
right nodes.
```shell
$ kubectl label --overwrite nodes <node name> role=loadbalancer
$ kubectl get nodes
NAME           LABELS                                                  STATUS    AGE
<node name>   kubernetes.io/hostname=<node name>,role=loadbalancer     Ready     2d
```

Now it's time to create the `loadbalancer`. If you wan't to use `https` you need
to create a `secret` and mount the volume in your pod template.
Here's my `loadbalancer-rc.yaml`:
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: service-loadbalancer
  namespace: kube-system
  labels:
    app: service-loadbalancer
    version: v1
spec:
  replicas: 1
  selector:
    app: service-loadbalancer
    version: v1
  template:
    metadata:
      labels:
        app: service-loadbalancer
        version: v1
    spec:
      nodeSelector:
        role: loadbalancer
      volumes:
      - name: ssl-volume
        secret:
          secretName: kodbasen-ssl-secret
      containers:
      - image: kubernetesonarm/loadbalancer
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8081
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        name: haproxy
        ports:
        # All http services
        - containerPort: 80
          hostPort: 80
          protocol: TCP
        # nginx https
        - containerPort: 443
          hostPort: 443
          protocol: TCP
        # mysql
        - containerPort: 3306
          hostPort: 3306
          protocol: TCP
        # haproxy stats
        - containerPort: 1936
          hostPort: 1936
          protocol: TCP
        # gogs ssh
        - containerPort: 2222
          hostPort: 2222
          protocol: TCP
        volumeMounts:
        - name: ssl-volume
          readOnly: true
          mountPath: "/ssl"
        resources: {}
        args:
        - --tcp-services=my-gogs-ssh:2222
        - --ssl-cert=/ssl/server.pem
        - --ssl-ca-cert=/ssl/ca.crt
```
He're you can see that I've exposed the `my-gogs-ssh` as a TCP service.

Now we're ready to expose our Gogs service to the outside world. We need to
change our Gogs service from
[Part 1]({{< relref "Gogs_and_Drone_On_Kubernetes-ARM_Part_1.md" >}}) slightly and add
some annotations.

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    serviceloadbalancer/lb.sslTerm: "true"
    serviceloadbalancer/lb.host: "gogs.replace.me"
    serviceloadbalancer/lb.cookie-sticky-session: "true"
  labels:
    app: my-gogs-service
  name: my-gogs-service
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app: my-gogs
  sessionAffinity: None
  type: ClusterIP
```

* `serviceloadbalancer/lb.sslTerm: "true"` annotation says that we wan't to
use `https`
* `serviceloadbalancer/lb.host: "gogs.replace.me"` is the `virtual host`
* `serviceloadbalancer/lb.cookie-sticky-session: "true"` enables sticky sessions
between your pods (`replicas > 1 `)

If everything is working you should be rewarded with a `ha-proxy` status page
where you can monitor your exposed services. Fire up
`http://<loadbalancer ip>:1936/` in your favorite browser and take a look.

That is all for now. In the next part we will take a look at the `CI` tool
[Drone](https://github.com/drone/drone) and how to get it working on our
_Kubernetes-ARM_ cluster. Prepare for a journey _down the rabbit hole_.
