+++
date = "2017-02-12T09:05:57+01:00"
title = "How I got started"
draft = true
tags = [ "Development", "Docker", "Kubernetes", "Weave", "Scaleway"]
categories = [ "Development", "Kubernetes", "Docker", "Cloud" ]
+++

Some times life takes a strange turn. Me writing this blog post for example. Two years ago I was looking for a
new platform for building e-services, for a former employer. The current solution was nothing more than a couple of web
servers. The solution had to be on-premise and I had pretty good insight in the Paas market and been looking at
CloudFoundry and the first generation of OpenShift. At the same time the great buzz about containers and Docker was
building up.

<!--more-->
I stumbled over Kubernetes and found out that the next version of OpenShift would be a total rewrite based on
Kubernetes. OpenShift was fairly new and the decision to do a total rewrite must have been a tough one. We were running
RHEL7 at work and there was an early version of Kubernetes available. This was in the summer of 2015 and I did some
experiments and wrote a report recommending that we should investigate further in solutions based on Docker and
Kubernetes.

2015 was a tough year for me personally and at the end of the year I had to take some time off. I was recommended long
walks and spending time doing something that I enjoyed. The thing is, I really enjoy building systems and I was really
excited about Kubernetes and Docker. I had also started writing short blog posts, to get some experience and for fun.

At the same time Ray Tsang and Arjen Wassink presented Kubernetes on Raspberry Pi at Devoxx Belgium and Lucas Källdström
had been working on the same thing at [kubernetes-on-arm](http://github.com/luxas/kubernetes-on-arm) that fall.
The [Hypriot](http://blog.hypriot.com/about/) team had done a great job of making Docker run on ARM and Raspberry Pi.

Sometimes things just fall in to place, when stars align… Here I was, with lots of spare time, a chance to do something
I really enjoy, so I wen’t down to our local Kjell & Company store (electronics store in Sweden) and bought some more
Pi:s, a switch and cables. I had this idea that if could build and run something on a Raspberry Pi based Kubernetes
cluster, then it would run and scale on any Kubernetes plattform. Think of it, that you can run the same software, on a
couple of $34 SoC boards, standing on your desktop, that is running in data centers serving millions of users around the
world.

There is also something very satisfying about building your own bare-metal cluster, with power supplies, network cables
and so on (plus it looks really awesome in the dark with all the flashing leds).

Short after building my first cluster I wrote my first post about running
[Kubernetes on ARM](http://larmog.github.io/2016/02/06/kubernetes-on-arm/). The writing has always been for fun (and for
the experience) and it gives my ideas a purpose and a goal. I always try to keep my posts short so that any potential
reader don’t get bored. I don't like to call what i'm writing *"How To"* articles or posts, I just want to share what
I've learned and my experience building things.

"Kubernetes on ARM" land hasn't always been easy and it would probably had been a lot easier just buying a couple of
*x86 Intel NUC*:s. But you want at least three nodes or more in your cluster and that makes it rather expensive and not
so available for everyone. I guess that the majority of us have a tight budget and funding is always a problem.

Since I started a little more than a year ago things have evolved enormously. We now have multi-architecture support in
Kubernetes and tools like `kubeadm` can help you get up and running in no time.

When I was trying to setup a Kubernetes cluster on Scaleway (bare-metal ARM servers cloud provider) I had the need of
encrypting the traffic between nodes in the cluster. On Scaleway you don’t have your own network so your traffic is
running unencrypted within their data center. I didn't know how to do that using `flannel` so I had a look at Weave Net
from Weaveworks. Weave Net has the option of encrypting traffic between nodes. The problem was there wasn't any ARM
version (a common problem in Kubernetes on ARM land) of weave-kube. So I made a small script that rebuilt the components
and images for ARM and it worked. Since then and with the latest release of Weave Net v1.9.0 there is now
multi-architecture support.

<img src="/media/wordcloud.png" style="float:right;height:200px">
So this is my story on how I got started writing blogs, mostly about Kubernetes on ARM. I’ve got lots of ideas in the
backlog and there is so much fun things to do and so little time. Thanks for reading my ramblings, hope you picked up
something new.

*Lars Mogren - The Nature of Software, Norrköping, Sweden*

#### Links

 * [Comparing OpenShift Enterprise 2 and OpenShift Enterprise 3](https://docs.openshift.com/enterprise/3.1/release_notes/v2_vs_v3.html)
 * [Running Kubernetes on a Raspberry Pi cluster](https://youtu.be/AAS5Mq9EktI)
 * [kubernetes-on-arm](http://github.com/luxas/kubernetes-on-arm)
 * [Hypriot](http://blog.hypriot.com)
 * [Minikube](https://github.com/kubernetes/minikube)
 * [`kubeadm`](https://kubernetes.io/docs/admin/kubeadm/)
 * [Scaleway](https://www.scaleway.com/)
 * [`flannel`](https://github.com/coreos/flannel)
 * [Weave Net](https://www.weave.works/products/weave-net/)


