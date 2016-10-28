+++
date = "2016-03-13T22:44:21+01:00"
draft = false
title = "ELK cluster on Kubernetes-On-ARM - Part 1"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "Elasticsearch", "Logstash", "Kibana", "ARM"]
categories = [ "Development", "Kubernetes", "Docker", "ARM", "Raspberry PI" ]
+++

One of the most important parts of running a cluster is to gain knowledge of
whats going on. Using tools like `kubectl logs` or `docker logs` is fine if you
run one or two nodes, but it soon gets impossible to get an overview of whats
going on, and you need to be able to view, query and monitor your logs from one
central place.
<!--more-->

This blog post is about setting up
_Elasticsearch + Logstash + Kibana ELK on Kubernetes-On-ARM_. You could question
the use of running _ELK_ on an _ARM_ cluster of _Raspberry Pi_:s, and you are
right, it is a bit to heavy. But I use the cluster as a way of learning, and the
steps are the same as if you are running on `x86_64` or in the cloud. There’s also
a bunch of new `ARM64` single-board computers on the way, _Raspberry Pi 3_
being the first.

You can use [`fluentd`](http://www.fluentd.org/) instead of `logstash` but we
stick to `logstash` for now.

Setting up _ELK_ on Kubernetes is nothing new, I'm using Paulo Pires:s
[kubernetes-elasticsearch-cluster](https://github.com/larmog/kubernetes-elasticsearch-cluster)
and [kubernetes-elk-cluster](https://github.com/larmog/kubernetes-elk-cluster),
modified for ARM.

The thing is, how do we collect our container logs and send them to logstash?
The first alternative is to use
[logstash-forwarder](https://github.com/elastic/logstash-forwarder) but the
project has been replaced by
[filebeat](https://github.com/elastic/beats/tree/master/filebeat), so that
becomes our second alternative. Then I found _gliderlabs_
[logspout](https://github.com/gliderlabs/logspout) project and _looplab_:s
[logspout-logstash](https://github.com/looplab/logspout-logstash) module, which
is a third alternative.

If you want to use `kubectl logs` or `docker logs` you
need to stick to the `json-file` or `journald` logging driver in Docker. The
`json-file` driver is rather resource consuming and not a great alternative for
production. There's a logstash plugin for
[`journald`](https://github.com/logstash-plugins/logstash-input-journald) that
looks promising and I hope to explore the different alternatives in the up
coming posts. Using `journald` for Docker means that we can use the same
mechanism for sending logs for hosts and containers.

To summarize what we want to achieve:
<div class="card-panel teal"><span class="white-text">
_We want to collect the logs and collect
them as quick as possible, but still be able to use the tools we're used to like
`docker logs` and `kubectl logs`._
</span></div>
We'll see how much of this we can achieve.

But we need to start somewhere, so let's get started with `logstash-forwarder`
and collect `syslog`.

#### Elasticsearch
First we need to get `elasticsearch` up and running. I have prepared all
necessary docker images so all you need to do is to deploy them to your cluster.

```shell
$ # Clone the git repo
$ git clone https://github.com/larmog/kubernetes-elasticsearch-cluster.git
$ cd kubernetes-elasticsearch-cluster
```
Follow Paulo's instructions to [deploy](https://github.com/larmog/kubernetes-elasticsearch-cluster#deploy)
the cluster. I recommend that you wait until each component is provisioned
before you deploy the next.

If all went well, you should now have `elasticsearch` running. You can check the
status using this command:
```shell
docker run --rm --dns=10.0.0.10 hypriot/armhf-busybox wget -q -O -  http://elasticsearch:9200/_cluster/health?pretty
```
#### Logstash + Kibana

Paulo’s solution uses the Lumberjack secure protocol, and you need generate your
own certificate and key for `logstash.default.svc.cluster.local`. I am using
`easyrsa3`. Make sure that you have valid `.key` and `.crt` files. You can of
course change your protocol to something other than `lumberjack` and make the
corresponding change in `logstash-forwarder`. I will show you how to setup
`lumberjack`.

Install Kelsey Hightower:s tool
[`conf2kube`](https://github.com/kelseyhightower/conf2kube). You will need it
later.

Create a temp working directory and copy your generated server key and
certificate:
```shell
$ mkdir logstash-tmp && cd logstash-tmp
$ cp xxx/<filename>.crt .
$ cp xxx/<filename>.key .
```

Create a `logstash.conf` file that corresponds to your key and certificate files.
```shell
$ echo <<< EOL
input {
  lumberjack {
    port => 5043
    ssl_certificate => "/logstash/certs/<filename>.crt"
    ssl_key => "/logstash/certs/<filename>.key"
    type => "lumberjack"
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
  }
}
EOL >> logstash.conf;
```
You shold now have three files: `<filename>.key <filename>.crt logstash.conf`

Create two secrets that will be mounted in the `logstash` pod
(see [`logstash-controller.yaml`](https://github.com/larmog/kubernetes-elk-cluster/blob/master/logstash-controller.yaml)).
```shell
$ # Create logstash-ssl secret
$ conf2kube -n logstash-ssl -f <filename>.crt | kubectl create -f -
$ kubectl patch secret logstash-ssl -p `conf2kube -n logstash-ssl -f <filename>.key`
$
$ # Create logstash.conf secret
$ conf2kube -n logstash.conf -f logstash.conf | kubectl create -f -
```

You should now have two `logstash` secrets:
```shell
$ kubectl get secrets
NAME                        TYPE                                  DATA      AGE
default-token-i5v41         kubernetes.io/service-account-token   2         59d
logstash-ssl                Opaque                                2         1d
logstash.conf               Opaque                                1         1d
```

Now clone the kubernetes-elk-cluster repo and create the logstash server and
kibana controller:
```shell
$ git clone https://github.com/larmog/kubernetes-elk-cluster.git
$ cd kubernetes-elk-cluster
$ kubectl create -f service-account.yaml
$ kubectl create -f logstash-service.yaml
$ kubectl create -f logstash-controller.yaml
$ kubectl create -f kibana-controller.yaml
```
Great! You should now have `logstash` and `kibana` running!

For you to be able to access `kibana`, you need to expose the service outside
your cluster. I assume that you have used the `Kubernetes-On-ARM` addon
`loadbalancer`. Kibana doesn't work with a sub-context so you need to expose
`kibana` as a virtual host.

```shell
$ # Run the following command with your Kibana host name.
$ sed -e 's/kibana.kodbasen.org/<hostname>/g' kibana-service.yaml > modified-kibana-service.yaml
$ kubectl create -f modified-kibana-service.yaml
```

If everything has gone according to plan you should see something similar:
```shell
$ curl -I http://<hostname>
HTTP/1.1 400 Bad Request
kbn-name: kibana
kbn-version: 4.4.0
content-type: application/json; charset=utf-8
cache-control: no-cache
Date: Sat, 12 Mar 2016 11:16:37 GMT
```
Never mind the `400` answer, you can see that Kibana has replied with
`kbn-name: kibana` and `kbn-version: 4.4.0`. Now try to open Kibana in your
favorite browser.

We have covered a lot of ground, and you should give your self a tap on the
shoulder. But were not done yet. We have _elasticsearch_ running and _logstash_
waiting for logs to be stored in elastic. _Kibana_ is hanging around waiting for
logs to appear in the `.kibana` index. But we need to start thinking of how
to forward logs from our nodes to _logstash_.

#### Logstash Forwarder

In this first part we will use `logstash-forwarder` to collect logs from our
nodes and forward them to logstash. In the following parts of this series we
will look into other ways. The `logstash-forwarder` project has been replaced
by `filebeat` but that´s a topic for the next post.

We need to run `logstash-forwarder` on every node that we wan't to collect logs
from. Kubernetes `1.2` will include daemon sets, which is a great feature for
running things on a set of nodes but we are still running `1.1.x`. In the
meantime, until `1.2` is released, we'll use static pods.

Repeat the following steps on every node where you wan't to collect logs:
```shell
$ mkdir -p /etc/logstash-forvarder/certs && mkdir /etc/logstash-forvarder/config
$
$ # Copy the CA certificate that was used to create the logstash cert.
$ cp ca.crt /etc/logstash-forvarder/certs
```

Create the `logstash-forwarder.conf` in `/etc/logstash-forvarder/config`:
```
{
  "network": {
    "servers": [
        "logstash.default.svc.cluster.local:5043"
    ],
    "timeout": 15,
    "ssl ca": "/logstash-forwarder/certs/ca.crt"
  },
  "files": [
    {
      "paths": [
        "/var/log/syslog"
      ],
      "fields": {"type": "syslog"}
    }
  ]
}
```

Create the `logstash-forwarder` static pod file `logstash-forwarder.json`:
```
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name":"logstash-forwarder"
  },
  "spec":{
    "containers":[
      {
        "name": "logstash-forwarder",
        "image": "larmog/logstash-forwarder:2.2.0",
        "imagePullPolicy": "IfNotPresent",
        "command": [
        ],
        "securityContext": {
          "privileged": true
        },
        "volumeMounts":[{
          "name": "certs",
          "mountPath": "/logstash-forwarder/certs"
        },
        {
          "name": "config",
          "mountPath": "/logstash-forwarder/config"
        },
        {
          "name": "syslog",
          "mountPath": "/var/log/syslog"
        }]
    }],
    "volumes": [
      {
        "name": "certs",
        "hostPath": {
          "path": "/etc/logstash-forwarder/certs"
        }
      },{
        "name": "config",
        "hostPath": {
          "path": "/etc/logstash-forwarder/config"
        }
      },
      {
        "name": "syslog",
        "hostPath": {
          "path": "/var/log/syslog"
        }
      }
    ]
  }
}
```
Copy the file to the manifest directory. In Kubernetes-On-ARM there located
here:

* On the master: `/etc/kubernetes/static-pods/master`
* On workers: `/etc/kubernetes/static-pods/worker`

The `kubelet` will pick up the static pod definition as soon as the file is
copied to the manifets directory. Run `docker ps|grep logstash-forwarder:2.2.0`
on the node, and you should see the container running.

And we are done! The `logstash-forwarder` starts processing events and if you
look at the container log, you should see something similar:
```
$ docker logs -f 8c4a3ba3f3e9
...
2016/03/11 15:13:41.575130 Registrar: processing 1024 events
2016/03/11 15:13:55.862568 Registrar: processing 1024 events
2016/03/11 15:14:01.671923 Registrar: processing 1024 events
2016/03/11 15:14:02.889663 Registrar: processing 256 events
```

#### To sum up

We have created a `elasticsearch` cluster for storing logs, and `logstash`for
processing incoming logs. We've also setup `kibana` for searching and
visualizing. On our nodes we have installed `logstash-forwarder`, to send all
`/var/log/syslog` events to `logstash`, using the `lumberjack` protocol.

In the next part in this series we will investigate how we can forward stdout
and stderr from our containers to `logstash`. As always, I hope you picked up
something new and found it worth while reading.

##### Docker images
All images are built from Pauls with generated `Dockerfile`:s for `armhf`.
The _Java_ base image for `elasticsearch` is
[larmog/armhf-alpine-java](https://hub.docker.com/r/larmog/armhf-alpine-java/)

##### Acknowledgement

* [Paulo Pires](https://github.com/pires) repositories for
`elasticsearch`, `logstash` and `kibana`.
* [Leanne Northrop](https://github.com/leannenorthrop) repository for Docker
Java image on `armhf`.

##### Links

* https://github.com/pires/kubernetes-elasticsearch-cluster
* https://github.com/pires/kubernetes-elk-cluster
* http://technologyconversations.com/2015/05/18/centralized-system-and-docker-logging-with-elk-stack/
* https://thepracticalsysadmin.com/shipping-logs-to-elk/
* http://www.dasblinkenlichten.com/logging-in-kubernetes-with-fluentd-and-elasticsearch/
* http://kubernetes.io/v1.1/docs/getting-started-guides/logging-elasticsearch.html
* http://kubernetes.io/v1.1/docs/user-guide/logging.html
* http://www.fluentd.org/guides/recipes/docker-logging
* https://github.com/GoogleCloudPlatform/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch/fluentd-es-image
* https://github.com/gliderlabs/logspout
* https://github.com/looplab/logspout-logstash
* https://hub.docker.com/r/gliderlabs/logspout/~/dockerfile/
* https://groups.google.com/forum/#!topic/google-containers/ZJzUIn_r3w4
* old https://github.com/stuart-warren/logstash-input-journald
* https://github.com/logstash-plugins/logstash-input-journald
* https://github.com/elastic/logstash/issues/1729
* https://github.com/vaijab/logstash-filter-kubernetes
* http://www.fluentd.org
