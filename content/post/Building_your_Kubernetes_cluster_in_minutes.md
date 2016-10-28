+++
date = "2016-07-14T19:16:43+02:00"
draft = false
title = "Building your Kubernetes cluster in minutes"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "ARM", "ARM64", "AMD64"]
categories = [ "Development", "Kubernetes", "Docker", "ARM", "Raspberry PI" ]

+++

With the release of Kubernetes `v1.3.0` there is now a cross platform way of
quickly setting up a Kubernetes cluster using Docker.
<!--more-->

[kube-deploy/docker-multinode](https://github.com/kubernetes/kube-deploy/tree/master/docker-multinode)
uses two instances of Docker daemon.


I've created a hybrid solution [Sloop](https://github.com/kodbasen/sloop) that uses kube-deploy/docker-multinode but runs
the kubelet *native*. The reason for running your `kubelet`:s *native* is the full support of [Persistent Volumes](http://kubernetes.io/docs/user-guide/persistent-volumes/).

Sloop uses the best of two worlds. Docker for bootstrapping your cluster and *native* kubelet for full Kubernetes functionality.

Give it a try and see what you think!
https://github.com/kodbasen/sloop
