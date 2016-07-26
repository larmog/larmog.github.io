+++
date = "2016-07-26T08:48:47+02:00"
draft = false
title = "Jenkins on Kubernetes ARM"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "ARM", "ARM64", "Jenkins"]
categories = [ "Development", "Kubernetes", "Docker", "CI/CD" ]
+++

<img src="https://www.arm.com/assets/images/ARM_Logo_Black.png" height="80">
<img src="https://secure.gravatar.com/avatar/26da7b36ff8bb5db4211400358dc7c4e.jpg" height="80">
<img src="https://raw.githubusercontent.com/kubernetes/kubernetes/master/logo/logo.png" height="80">
<img src="https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png" height="80">

[Jenkins](https://jenkins.io/) is a tool for *Continuous Integration and Continuous Delivery* and has a plugin for provisioning builds (slaves) cross a Kubernetes cluster.
<!--more-->

The plugin kicks off every build in a new `pod` and tears it down when the build is complete. This means that every build is clean and reproducible. This is clearly something that we want in *Kubernetes-On-ARM*.

##### Before we begin

There is now support for multiple platforms with the relase of Kubernetes `v1.3.0`, `arm` and `arm64` being two of them. There is also a side project: [`kube-deploy/docker-multinode`](https://github.com/kubernetes/kube-deploy/tree/master/docker-multinode) that lets you setup a Kubernetes cluster in minutes, using docker.

[Sloop](https://github.com/kodbasen/sloop) - is my project that uses `kube-deploy/docker-multinode` for running Kubernetes *"native"*. You can read more about it in an earlier post [Building your Kubernetes cluster in minutes]({{< relref "Building_your_Kubernetes_cluster_in_minutes.md" >}}). You're more than welcome to check(it)out and give it a try.

###### Prerequisites

* A Kubernetes `v1.3.0` cluster with `arm` or `arm64` nodes.

##### Setting up Jenkins

The Jenkins master keeps state and needs a durable volume for storing
data about your builds. I highly recommend using something other than `emptyDir` or `hostPath`. I use a `nfs` volume.

I have created a Jenkins image: [`kodbasen/jenkins-arm`](https://hub.docker.com/r/kodbasen/jenkins-arm/) and here's a `deployment` you can use:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
  labels:
    run: jenkins
  name: jenkins
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: jenkins
  template:
    metadata:
      labels:
        run: jenkins
    spec:
      containers:
      - image: kodbasen/jenkins-arm:2.7.1
        imagePullPolicy: IfNotPresent
        name: jenkins
        ports:
        - containerPort: 8080
          protocol: TCP
          name: web
        - containerPort: 50000
          protocol: TCP
          name: slaves
        volumeMounts:
        - mountPath: /var/jenkins_home
          name: jenkinshome
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: jenkins
```

I'm using a `persistentVolumeClaim:` named `jenkins`. I leave it up to you to sort out your volume configuration.

Next we need to expose our Jenkins deployment as a `service`:

```
apiVersion: v1
kind: Service
metadata:
  labels:
    run: jenkins
  name: jenkins
  namespace: default
spec:
  ports:
  - name: web
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: slaves
    port: 50000
    protocol: TCP
    targetPort: 50000
  selector:
    run: jenkins
```

You should now have Jenkins up and running. Now check your Jenkins `pod` log and copy the *first time password* and head over to Jenkins to set it up, and while your there, install the Kubernetes plugin. Come back here when your done and I'll show you how to setup the plugin.

##### Setting up Jenkins Kubernetes plugin

The Kubernetes plugin uses a `slave` image and a `pod`-template so there´s
not much for you to configure. Follow the instruction on the plugin [page](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) but use [kodbasen/jenkins-slave-arm](https://hub.docker.com/r/kodbasen/jenkins-slave-arm/) image as *Docker image*.
{{< img src="/media/kubernetes-pod-template.png" title="Kubernetes Pod Template" >}}

In your build configuration you need to restrict where the build can be run. Set it to use your **pod template label**.
{{< img src="/media/label-expression.png" title="Restrict where the project can be run" >}}

##### Running Docker builds

If you wan't to run docker builds on Jenkins theres another slave image that you can use: [kodbasen/jenkins-docker-slave-arm](https://hub.docker.com/r/kodbasen/jenkins-docker-slave-arm/). This image includes the docker binaries and runs docker builds on the host where the pod is running. For this to
work you need to mount three `hostPath` volumes:
{{< img src="/media/pod-volumes.png" title="Pod template volumes" >}}

##### Running your builds

When running your build you'll se the build pending while the slave is provisioned.
{{< img src="/media/pending-build.png" title="Pending build" >}}

You can control which nodes should be used for running builds using a `nodeSelector`.

##### Conclusion

I've started moving all my builds from Drone over to Jenkins. If you decide to give Jenkins a try, and you're using [Sloop](https://github.com/kodbasen/sloop), let me know how it goes.

I hope you picked up something new, and that it was worth while reading this post.
