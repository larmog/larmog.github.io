+++
date = "2016-02-13T11:50:12+01:00"
draft = true
title = "Dashboard on Kubernetes ARM"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI" ]
categories = [ "Development", "Docker" ]
+++

[Lucas Käldström](https://github.com/luxas) added the new __Dashboard__ addon
in the `dev` bransch to [Kubernetes On ARM](https://github.com/luxas/kubernetes-on-arm)
and I decided to try it out.
<!--more-->

Here's how to install it:

```shell
$ mkdir /etc/kubernetes/source/addons/dashboard
$ curl -sSL https://raw.githubusercontent.com/luxas/kubernetes-on-arm/dev/addons/dashboard.yaml > \ /etc/kubernetes/source/addons/dashboarddashboard.yaml
$ kube-config enable-addon dashboard
```

And here it is:
{{< img src="/media/dashboard.jpg" title="Kubernetes Dashboard on ARM" >}}

Cool, Thanks [Lucas](https://twitter.com/kubernetesonarm)!
