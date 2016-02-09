+++
date = "2016-02-08T16:39:34+01:00"
draft = true
title = "Gogs and Drone On Kubernetes-ARM - Part 3"
author = "Lars Mogren"
tags = [ "Development", "Docker", "Kubernetes", "Raspberry PI", "Gogs", "Drone"]
categories = [ "Development", "Kubernetes", "Docker" ]

+++

This is third part in a series on setting up Gogs and Drone on Kubernetes-ARM.
Here's the earlier parts:
* Part 1
* Part 2

In this third part I will try to show how to setup Drone on Kubernetes-ARM.
A disclaimer, I'm new to Drone and out on deep water so...

If your wondering about Drone take a look at the
[GitHub page](https://github.com/drone/drone).

_"Every build is executed inside an ephemeral Docker container, giving
developers complete control over their build environment with guaranteed
isolation."_

This means that the `build agent` is a container and all the plugins are also
containers. This makes it a bit problematic running Drone on ARM because we
need to rebuild everything, from Drone it self to all plugins that we wan't to
use. We need to setup a Drone infrastructure for ARM.
