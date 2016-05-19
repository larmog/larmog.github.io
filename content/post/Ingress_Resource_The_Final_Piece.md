+++
date = "2016-05-19T13:29:40+02:00"
draft = false
title = "Ingress Resources, the final piece of the puzzle?"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "ARM"]
categories = [ "Development", "Kubernetes", "Docker" ]
+++

I've been using service-loadbalancer in my cluster and annotated my services to
make them available outside the cluster network. With the release of Kubernetes
1.2 there is another alternative, the Ingress Resources.

You can read more about the Ingress resource in the [Kubernetes Reference Documentation](http://kubernetes.io/docs/user-guide/ingress/).
<!--more-->

I've always felt that there's been something missing in Kubernetes when it comes
to exposing services. There has been multiple ways of doing it, and it's
been up to you or your Kubernetes provider to decide how. You can use the
`service-loadbalancer` method but it's a bit messy. You need to keep
track of your annotations, and I personally think that they shouldn't be there
in the first place. With the Ingress Resource you get a clear separation between
your service, and how you expose the service outside your cluster.

#### Ingress controller

Creating an Ingress without an Ingress controller will have no effect. So we
first need to install a controller. You can read more about Ingress controllers
[here](https://github.com/kubernetes/contrib/tree/master/ingress/controllers).

There is a `nginx-ingress-controller` in the
[contrib](https://github.com/kubernetes/contrib) repository and an image
`gcr.io/google_containers/nginx-ingress-controller`. If you read any of my
earlier [posts]({{< relref "2016-02-05-Kubernetes-On-ARM.md" >}}), then you know
that I'm running [`Kubernetes-On-ARM`](https://github.com/luxas/kubernetes-on-arm).

There is no official `nginx-ingress-controller-arm` image yet, but if you want
you can use mine. I've built the same image chain as the
`nginx-ingress-controller`, but for `armhf`. Here's the image chain:

* https://hub.docker.com/r/kodbasen/ubuntu-slim-armhf/
* https://hub.docker.com/r/kodbasen/nginx-slim-armhf/
* https://hub.docker.com/r/kodbasen/nginx-ingress-controller-armhf/

I've also built the `defaultbackend-armhf` image:
https://hub.docker.com/r/kodbasen/defaultbackend-armhf/

All you need to do is to change the images in the `rc.yaml` file:
```shell
wget -q https://raw.githubusercontent.com/kubernetes/contrib/master/ingress/controllers/nginx/rc.yaml \
&& sed -i -e "s;gcr\.io/google_containers/nginx-ingress-controller;kodbasen/nginx-ingress-controller-armhf;" "rc.yaml" \
&& sed -i -e "s;gcr\.io/google_containers/defaultbackend;kodbasen/defaultbackend-armhf;" "rc.yaml"
```

Now your ready to deploy your `nginx-ingress-controller-armhf`:
```shell
$ kubectl create -f rc.yaml
```

That's it, you can now start experimenting with **Ingress Resources** on
[`Kubernetes-On-ARM`](https://github.com/luxas/kubernetes-on-arm).

Ingress Resources is maybe the final piece of the puzzle, to make Kubernetes
totally awesome. What do you think?
