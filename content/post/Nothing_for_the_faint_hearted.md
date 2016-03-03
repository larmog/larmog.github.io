+++
date = "2016-03-03T09:12:49+01:00"
draft = false
title = "Nothing for the faint hearted"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "GlusterFS", "Gogs", "Drone", "ARM"]
categories = [ "Development", "Kubernetes", "Docker" ]
+++

I’ve been running my _Kubernetes-On-ARM_ cluster a couple of weeks now and I’ll
try to summarize my experiences. If you followed my earlier posts, then you know
that I'm running [Gogs](https://gogs.io/) and
[Drone](https://github.com/drone) on a Kubernetes cluster with
[GlusterFS](https://www.gluster.org/) for storing. I’m using [Kubernetes-On-ARM](https://github.com/luxas/kubernetes-on-arm)
on [HypriotOS](http://blog.hypriot.com/).

In this post I will try to summarize my experiences so far.
<!--more-->

##### Dind and Kubelet does not play well together

One of the reasons that I wanted to run Drone was it’s use of Docker, and to
build and push docker images to a registry. Drone has a plugin that does all
that and uses `dind` (Docker in Docker).

The examples and [documentation](https://github.com/drone-plugins/drone-docker/blob/master/DOCS.md)
is rather straightforward, nothing complicated. You need to install and setup
the [client](http://readme.drone.io/devs/cli/), and setup your
[secrets](http://readme.drone.io/usage/secrets/) so that you don’t store your
username and password in your build file.

There was only one problem, it didn't work …

I could not get the plugin to work. After turning on `DEBUG`:
```
# .drone.yml
...
publish:
  docker:
    environment:
    - DOCKER_LAUNCH_DEBUG=true
...
```
I could see this error message:
{{< img src="/media/drone-log.jpg" title="Drone Docker plugin DEBUG" >}}

It gave me a real headache and finally, after some googling, I found these
issues:

* https://github.com/drone/drone/issues/1352
* https://github.com/kubernetes/kubernetes/issues/20671
* https://github.com/kubernetes/kubernetes/issues/18202

The `kubelet` is causing the `/sys/fs/docker` file-system to becoming read-only,
this will hopefully be resolved in Kubernetes 1.2. Meantime, the solution is,
to remove one of the nodes from the Kubernetes cluster and use it as a remote
worker-node in Drone.

##### Containerized Kubelet

There are issues, when running the `kubelet` in a container, and is work in
progress, to resolve those issues.

* https://github.com/kubernetes/kubernetes/issues/4869
* https://github.com/kubernetes/kubernetes/issues/14403

##### Glusterfs USB and 100Mbit/Ethernet network

I’ve been wanting to try out [Go CD](http://go.cd) for some time now, and try to set up a pipeline on Kubernetes. If you take a look at the [system requirements](https://docs.go.cd/current/installation/system_requirements.html) for Go CD, you will see that this is not suitable for a Raspberry Pi. The Go Server is a giant monolith, but what the heck, sometimes you need to check out the limit. I created a Docker image for ARM and tried to start up a pod.

I got the machine started, but it took an hour. Most of the cpu-time during
start was spent on `glusterfs`. The _Go CD_ server uses a lot of I/O during
start and I measured the read/write speed on the glusterfs to around 1,5-2 MB/s.
One should not expect any great performance when running `glusterfs` on
_Raspberry Pi_. There are some real bottlenecks on network and USB.

##### Upgrading to Docker v1.10.2

The _Hypriot_ team has written a post about [Test, build and package Docker for ARM the official way](http://blog.hypriot.com/post/test-build-and-package-docker-for-arm-the-official-way/).
That was one of reasons why I wanted to upgrade Docker to `v1.10.2`.
But has shown to have some issues. The `kubectl exec` command doesn’t work
anymore and is reported here:

https://github.com/kubernetes/kubernetes/issues/20884

and is a duplicate of:

* https://github.com/kubernetes/kubernetes/issues/19720
* https://github.com/fsouza/go-dockerclient/issues/455

##### Summarize

Running _Kubernetes-On-ARM_ is nothing for the _faint hearted_. There are some
issues that hopefully will be resolved in a near future. I’m still very pleased
with the cluster and it’s a great environment for building _microservices_ and
validating your system architecture. For instance, it’s not possible to build a
big monolith. If your system runs and scales on Kubernetes-On-ARM
(built on Raspberry Pi) then you can be sure of that it runs and scales on any
platform.
