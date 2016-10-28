+++
date = "2016-10-28T05:15:54+02:00"
draft = false
title = "Installing Kubernetes on ARM with kubeadm"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "Odroid", "ARM"]
categories = [ "Development", "Kubernetes", "Docker", "ARM", "Raspberry PI" ]
+++

The tool `kubeadm` is an awesome tool for getting started with Kubernetes.
This short post shows you how to get started on `Raspberry Pi 3`.
<!--more-->

##### OS

I've used HypriotOS ["Blackbeard"](http://blog.hypriot.com/post/releasing-HypriotOS-1-0/)
when using `Raspberry PI 3` and [Odrobian](http://oph.mdrjr.net/odrobian/doc/getting-started.html) for `Odroid C2`.

For some reason `cgroup=cpuset` wasn't enabled in the kernel for HypriotOS so
I had to enable it in `/boot/cmdline.txt`.

```shell
HypriotOS/armv7: root@minion1 in ~
$ cat /boot/cmdline.txt
+dwc_otg.lpm_enable=0 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 cgroup_enable=cpuset cgroup_enable=memory swapaccount=1 elevator=deadline fsck.repair=yes rootwait console=ttyAMA0,115200 kgdboc=ttyAMA0,115200
```

##### Installing

The guide [Installing Kubernetes on Linux with kubeadm](http://kubernetes.io/docs/getting-started-guides/kubeadm/) shows
you how to install, if you scroll down to the end of the page (which I didn't
and wish I had) you can see how to install Kubernetes using `kubeadm` for `ARM`.
Thank's to [Lucas](https://github.com/luxas) I guess.

Also while reading the official documentation
for [kubeadm](http://kubernetes.io/docs/admin/kubeadm/), there's
a `kubeadm init` flag `--use-kubernetes-version` you should set to
at least `v1.4.3`.

##### Using `weave-kube`

I didn't read the last part of the guide (or it wasn't present while I read it)
so I had to do things the hard way (or backwards). The good thing is that
you can use my `ARM` version of [weave-kube](https://github.com/weaveworks/weave-kube)
instead of flannel, for pod networking.

```shell
kubectl create -f https://raw.githubusercontent.com/kodbasen/weave-kube-arm/master/weave-daemonset.yaml
```

##### Conclusion

`kubeadm` is awesome :)
