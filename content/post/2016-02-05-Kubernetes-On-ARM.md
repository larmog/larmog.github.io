+++
date = "2016-02-06T10:33:08+01:00"
draft = false
title = "Kubernetes On ARM"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI" ]
categories = [ "Development", "Docker" ]
+++

I really like Kubernetes for orchestrating Docker containers. If you're don't
familiar with Kubernetes I can highly recommend to take a look at
(http://kubernetes.io).
<!--more-->

You can easily run you're own Kubernetes cluster on your local machine using
Vagrant or run Kubernetes in the cloud using AWS, Azure or Google Compute.
But I like a more hands on solution and I always wanted my own "data center".
On the other hand I don't want to spend a fortune building a DC just for fun.

A great alternative is to use ARM SoC boards like Raspberry PI. Thank's to
[Lucas Käldström](https://github.com/luxas) and
[Hypriot](http://blog.hypriot.com/) this is a rather straight forward process.

I used the `hypriotos` image and Lucas `deb`-package to install `kube-config`.

{{< img src="/media/IMG_1936.png" title="Six Raspberry Pi:s in a cluster" >}}

{{< img src="/media/k8s.png" title="Nodes in my cluster" caption="Here you can see the nodes in the cluster. I'm using the service-loadbalancer addon (a topic for another post)." >}}
