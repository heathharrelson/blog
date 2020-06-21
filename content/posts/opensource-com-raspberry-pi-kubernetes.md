---
title: Opensource.com Raspberry Pi Kubernetes Cluster Article
description:
date: 2020-06-20T22:22:58.594-07:00
draft: false
tags:
- kubernetes
- raspberry pi
---

As often happens, after spending several evenings getting [my Raspberry Pi 4 Kubernetes cluster]({{< ref "raspberry-pi-cluster-kubeadm" >}}) working, I found this similar [Opensource.com article on how to build a Raspberry Pi Kubernetes cluster](https://opensource.com/article/20/6/kubernetes-raspberry-pi).<!--more--> The article is strikingly similar, although more of the setup is manual. The main difference is that the article uses Ubuntu for the Raspberry Pi's operating system to take advantage of the availability of arm64 Docker images.

A lot of my effort was spent on automating machine setup and dealing with firewall issues, so finding this might or might not have saved me any time. Still, I'm glad to see more articles on the topic, which hopefully will make it easier for others to get started on their own clusters.

[Opensource.com: Build a Kubernetes cluster with the Raspberry Pi](https://opensource.com/article/20/6/kubernetes-raspberry-pi)

