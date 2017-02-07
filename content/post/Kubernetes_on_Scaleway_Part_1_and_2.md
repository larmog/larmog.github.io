+++
date = "2016-10-09T22:00:00+02:00"
draft = false
title = "Kubernetes on Scaleway - Part 1 (Revisited) & 2"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Traefik", "Scaleway"]
categories = [ "Development", "Kubernetes", "Docker", "Cloud" ]
+++

<a href="http://scaleway.com">
<img src="https://www.scaleway.com/img/scaleway.f0e4.svg" style="float:right;height:80px"></a>

Things doesn't always turn out as planned. This is specially true when it comes
to this series, on how to setup *Kubernetes* on *Scaleway*. The plan was that
part 2 would be about setting up an `ingress-controller` and securing the
`api-server` and `dashboard`. But in the meantime, while I was struggling with
getting `weave` working on `ARM` and re-writing my own little project `sloop`,
Kubernetes `v1.4.0` was released with the `kubeadm` tool.
<!--more-->

With the tool `kubeadm`, you can easily install a secure Kubernetes cluster. It
makes projects like my own `Sloop` unnecessary, and that's a good thing.

In this post I will show how to setup Kubernetes using `kubeadm` and setting up
an `ingress-controller` and how to secure your `api-server` and `dashboard`. We
won't be using `ARM` for this, that's a topic for another blog post.

Here's the recipe:
```
1 Scaleway account
2 VC1S servers
1 wildcard certificate
2 kubeadm
1 DaemonSet weave-kube CNI plugin
1 traefik reverse proxy (http://traefik.io)
1 traefik ingress-controller
1 domain name
1 DNS A record
```
You can replace the domain and `DNS A` record with entries in your `hosts` file
if you just want to try this out.

Let's get started!

#### Create master node

Creating your Kubernetes cluster using `kubeadm` is described excellent in this
[guide](http://kubernetes.io/docs/getting-started-guides/kubeadm/). I only
include the steps here for completeness.

First thing is to create the `k8s-master` node:

```shell
# Create your master node:
$ scw start -w $(scw create --name="k8s-master" --commercial-type="VC1S"  Docker)
$ scw ps
```

Login to your `k8s-master` node and install the Kubernetes components:

```shell
$ curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
$ cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
$ apt-get update
$ apt-get install -y kubelet kubeadm kubectl kubernetes-cni
$ kubeadm init
```

#### Create your worker node

Repeat the steps from installing the `k8s-master` but name the node: `k8s-node1`
and use the `join` command:

```shell
$ kubeadm join --token <token> <k8s-master IP>
```

#### Install a pod network

```shell
$ kubectl apply -f https://git.io/weave-kube
daemonset "weave-net" created
```

If you run `kubectl get nodes`, on your master, you should now see two nodes:

```shell
$ kubectl get nodes
NAME         STATUS    AGE
k8s-master   Ready     1d
k8s-node1    Ready     1d
```

Install the latest `dashboard` and find the service ip:
```shell
$ kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
$ kubectl get svc -n kube-system | grep dashboard
kubernetes-dashboard   100.xx.xx.xxx    <nodes>       80/TCP          2h
```

We will be using the `dashboard` service ip later on. That was easy, don't you
think? You now have your own Kubernetes cluster running.

#### Using `traefik` as `ingress-controller` and reverse proxy

Using `CNI` for the pod network comes with a price tag. You can't use `hostPort`
in your pods right know. Im sure this will be fixed in later versions but right
now that's a problem. A `ingress-controller` uses `hostPort` to expose the
cluster.

The solution I found is to use a reverse proxy running as a static pod on one
of the nodes (I choose `k8s-node1`) and configuring it with `hostNetwork: true`.
Traefik is a reverse proxy written in Go and we will be using it for securing
the `dashboard` with basic authentication and for proxying request to the
`ingress-controller` which happens also be `traefik`.

Let's start with installing `traefik` as `ingress-controller`:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    k8s-app: traefik-ingress-lb
  name: traefik-lb
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: traefik-ingress-lb
    spec:
      containers:
      - args:
        - --kubernetes
        image: traefik:v1.0.0
        imagePullPolicy: IfNotPresent
        name: traefik-lb
        ports:
        - containerPort: 80
          name: web
          protocol: TCP
```

As you can see there's no `hostPort`. Next step is to expose the `traefik-lb`
as a service:

```
$ kubectl expose deployment traefik-lb --port=80 --target-port=80 -n kube-system
$ kubectl get svc -n kube-system | grep traefik-lb
traefik-lb          100.xx.xxx.xxx   <none>        80/TCP          1d
```

We now have two services `kubernetes-dashboard` and `traefik-lb`. These are
the `backends` for our `traefik` reverse proxy.

Here's the `traefik.toml` file for our proxy:

```
logLevel = "DEBUG"
defaultEntryPoints = ["http", "https"]

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      CertFile = "/etc/traefik/server.crt"
      KeyFile = "/etc/traefik/server.key"
  [entryPoints.https2]
  address = ":8443"
    [entryPoints.https2.tls]
      [[entryPoints.https2.tls.certificates]]
      CertFile = "/etc/traefik/server.crt"
      KeyFile = "/etc/traefik/server.key"
    [entryPoints.https2.auth.basic]
    users = [ "test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/" ]

[web]
address = ":6443"
CertFile = "/etc/traefik/server.crt"
KeyFile = "/etc/traefik/server.key"
ReadOnly = true
  [web.auth.basic]
  users = [ "test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/" ]

[file]
filename = "/etc/traefik/rules.toml"
watch = true
```

Save the file to `/etc/traefik` directory on the `k8s-node1`.
Use `htpasswd` to generate a user name and password and edit the file. Change
the `CertFile:` and `KeyFile:` to point to your wildcard certificate and key.
We're using three `entryPoints`. Port `80` and `443` is reserved for our
services in the cluster, `8443` is used for the `kubernetes-dashboard` and port
`6443` is used for the `traefik webui`.

Next we need to configure traefik `frontend`:s and `backend`:s in a configuration
file: `/etc/traefik/rules.toml`

```
[frontends]

  [frontends.frontend1]
  passHostHeader = true
  backend = "dashboard-backend"
  entrypoints = ["https2"]
    [frontends.frontend1.routes.test_1]
    rule = "Host:dashboard.mydomain.com"

  [frontends.frontend2]
  passHostHeader = true
  backend = "k8s"
    [frontends.frontend2.routes.default]
    rule = "HostRegexp:{subdomain:[a-z]+}.mydomain.com"

[backends]
  [backends.dashboard-backend]

    [backends.dashboard-backend.LoadBalancer]
    method = "wrr"

    [backends.dashboard-backend.servers.server1]
    url = "http://100.xx.xx.xxx:80"

   [backends.k8s]

     [backends.k8s.LoadBalancer]
     method = "wrr"

     [backends.k8s.servers.server1]
     url = "http://100.xx.xx.xxx:80"
```
Change the ip of the `backends.dashboard-backend.servers.server1` `url` to your
`kubernetes-dashboard` service and change the ip of the
`backends.k8s.servers.server1` `url` to the `traefik-lb` service. You also need
to edit the domain so that it matches your wildcard certificate.

Now it's time to create our static pod. Copy the following to
`/etc/kubernetes/manifests/traefik.yaml`:

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: traefik
  name: traefik
  namespace: kube-system
spec:
  containers:
  - image: traefik:v1.1.0-rc1
    imagePullPolicy: IfNotPresent
    name: traefik
    resources: {}
    volumeMounts:
    - mountPath: /etc/traefik
      name: config
      readOnly: false
    ports:
    - name: http
      containerPort: 80
      hostPort: 80
      protocol: TCP
    - name: https
      containerPort: 443
      hostPort: 443
      protocol: TCP
    - name: webui
      containerPort: 6443
      hostPort: 6443
      protocol: TCP
    - name: dashboard
      containerPort: 8443
      hostPort: 8443
      protocol: TCP
  hostNetwork: true
  restartPolicy: Always
  securityContext: {}
  volumes:
  - name: config
    hostPath:
      path: /etc/traefik
```

The final step is to create a DNS A record pointing to the `k8s-node1`:s public  
IP.

Time to check if everything is working:
```
$ # The dashbord requires authentication
$ curl -s https://dashbord.mydomain.com:8443
401 Unauthorized
$
$ # The traefik webui also requires authentication
$ curl -s https://webui.mydomain.com:6443
401 Unauthorized
$
$ # All other requests on port 80 and 443 is forward to k8s and traefik-lb
$ curl -s http://anyapp.mydomain.com.org
404 page not found
$ curl -s https://anyapp.mydomain.com.org
404 page not found
```
Ok, it seems to work. I leave it up to you to explore the `dashboard` and
`traefik` `webui` in you favorite browser.

#### Conclusion

We now got a secured Kubernetes cluster that we can use for hosting all our apps
and services. In the last post in this series we will setup `PersistentVolume`:s
using *GlusterFS*.
