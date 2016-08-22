+++
date = "2016-08-17T20:48:11+02:00"
draft = false
title = "Setting up Kubernetes on Scaleway - Part 1"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Sloop", "ARM", "AMD64", "Scaleway"]
categories = [ "Development", "Kubernetes", "Docker, Cloud" ]
+++

<a href="http://scaleway.com">
<img src="https://www.scaleway.com/img/scaleway.f0e4.svg" style="float:right;height:80px"></a>

This post is the first in a series on setting up Kubernetes using the bare-metal cloud provider Scaleway. Scaleway offers bare-metal servers both for ARM and x86-64. I will show how to
set up Kubernetes on ARM using the Scaleway C1 server.
<!--more-->

##### Disclaimer

This work is done to the best of my ability, but you should know that i'm new to Scaleway and i'm writing this in my spare time, and sometimes in a haste. See it as an inspiration. Your feedback is always welcome.

##### Creating servers

First you need to sign up for a Scaleway account. Then head over to the **Servers** page and create two C1 servers. Select the Docker image from *ImageHub* and make sure you assign a public IP for both servers.

{{< img src="/media/scw-c1-server.png" title="Creating a C1 Docker ready server for Kubernetes." >}}

##### Fixing missing link

I had some problems with the dynamic link interpreter for the `hyperkube` binary, but creating this link on both servers fixes the problem.

```shell
root@scw-xxxxx:~# cd /lib
root@scw-xxxxx:/lib# ln -s arm-linux-gnueabihf/ld-2.23.so ld-linux.so.3
```

##### Cloning Sloop

You know what Sloop is right? If not take a quick look at this earlier post: [Building your Kubernetes cluster in minutes]({{< relref "Building_your_Kubernetes_cluster_in_minutes.md" >}}).

Clone the [Sloop](https://github.com/kodbasen/sloop) project on both servers and
create a `sloop.conf` file in the Sloop directory that looks like this:

```bash
K8S_VERSION=v1.3.5
FLANNEL_NETWORK=10.10.0.0/16
FLANNEL_BACKEND=host-gw

# This entry is only required on your workers
MASTER_IP=<your master server Public-IP>
```
`MASTER_IP` should only be set on your worker(s).

##### Securing your environment

Scaleway offers _Security_ for your network through static IP package inspection. Your rules are configured in a _security group_. The problem is that you can't have a default rule that matches and blocks all inbound traffic last in the list, and only add specific ports.

You need to add rules for all ports you want to drop traffic on and add your server to the top of the list with a rule that accepts all traffic.
{{< img src="/media/scw-security-group-rules.png" title="Security Group rules" >}}

##### Starting your master

Now it's time to start up your master:

```shell
root@scw-master:~/sloop# ./sloop-master.sh
```
You can verify that your master is running using `kubectl`:

```shell
root@scw-master:~# kubectl get nodes
NAME          STATUS    AGE
10.1.xxx.xxx   Ready     1m
root@scw-master:~#
root@scw-master:~#
root@scw-master:~# kubectl cluster-info
Kubernetes master is running at http://localhost:8080
KubeDNS is running at http://localhost:8080/api/v1/proxy/namespaces/kube-system/services/kube-dns
kubernetes-dashboard is running at http://localhost:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
```

##### Starting your worker(s)

Now it's time to bring up your worker:

```shell
root@scw-worker:~/sloop# ./sloop-worker.sh
```

If you switch back to your `scw-master` server and run `kubectl get nodes` you
should see two nodes:

```shell
root@scw-master:~/sloop# kubectl get nodes
NAME          STATUS    AGE
10.1.xxx.xxx   Ready     1d
10.1.xxx.xxx   Ready     1d
```

##### Verify that everything works

Your `api-server` is unprotectd and we added a rule to our _Security Group_ to block all trafic on port `8080`, but you can verify that everything works by accessing the
`dashboard` on `http://<MASTER-PUBLIC-IP>:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard`, if you remove the rule.

##### Conclusion

We now have 2 nodes running Kubernetes on __ARM__. In the next post in this series i hope to show how to setup an `ingress-controller` with certificates and how to secure your api-server.
