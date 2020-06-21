---
title: "Building a Raspberry Pi Kubernetes Cluster with Kubeadm"
description: Describes how to set up a production-like Kubernetes cluster using Raspberry Pi 4 model B computers and the kubeadm utility.
date: 2020-06-19T22:21:38.051-07:00
draft: false
tags:
- kubernetes
- raspberry pi
---

This article describes how to build a personal [Kubernetes](https://kubernetes.io/) cluster using [Raspberry Pi](https://www.raspberrypi.org/) single-board computers and [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/).<!--more-->
<!-- There is a TODO LINK companion article that describes how to do the same thing using [k3s](https://k3s.io/). -->

## Materials

### Hardware

I used the following hardware to build the cluster described below.

* 3 Raspberry Pi 4 Model B (4 GB)
* 3 LoveRPi PoE hat for Raspberry Pi 4 Model B (compact)
* TP-Link 8 port PoE gigabit switch

I previously built a cluster using Raspberry Pi 3 Model B+ computers. While this worked, the performance was underwhelming, and it didn't leave many resources left for applications. If you have the option, the Raspberry Pi 4 is much more capable.

Using [power over Ethernet (PoE)](https://en.wikipedia.org/wiki/Power_over_Ethernet) is optional. You can use pretty much any Ethernet switch to build your cluster, but with PoE the result is much tidier.

### Software

* [Hypriot 1.12.2](https://github.com/hypriot/image-builder-rpi/releases/tag/v1.12.2)
* [Kubernetes 1.18.3](https://github.com/kubernetes/kubernetes/releases/tag/v1.18.3)
* [Weave Net 2.6.5](https://github.com/weaveworks/weave/releases/tag/latest_release)

Using [Hypriot](https://blog.hypriot.com/downloads/) for the operating system saves installation time, because Docker is preinstalled. However, Hypriot is currently built for 32-bit armv7 architecture. The Raspberry Pi 3 and 4 can run 64-bit binaries, and plenty of the Docker containers available are only built for arm64. If that matters to you, you might want to choose a different OS.

### Networking

The Ethernet interfaces are used for cluster networking and have static IP addresses in the 10.1.1.0/24 range. WiFi interfaces are used for internet access and are configured using DHCP.

## Setup

### Hypriot Config

Hypriot allows configuring the disk images using [cloud-init](https://cloudinit.readthedocs.io/en/20.2/index.html), which reduces the manual steps required to get the cluster up and makes setup much more repeatable.

#### user-data

Following [`user-data` examples from the Hypriot GitHub repo](https://github.com/hypriot/flash/tree/master/sample), I created a [`user-data` file](https://cloudinit.readthedocs.io/en/20.2/topics/examples.html#yaml-examples) for each host. See [this Gist](https://gist.github.com/heathharrelson/ffde93b78d9a1ecd77a47e4f8c16d297) for a full example.

This `user-data` file:

* sets the hostname
* changes the username from `pirate` to `k8s`
* adds an authorized SSH key
* adds a static `/etc/hosts` file
* configures the `wlan0` and `eth0` interfaces
* sets up a workaround for issues with `iptables-nft`
* adds the Kubernetes Apt repo

#### network-config

I modified `network-config` to disable the default configuration, which enables DHCP on `eth0`.

```yaml
version: 1
config: disabled
  # - type: physical
  #   name: eth0
  #   subnets:
  #     - type: dhcp
```

## Installation

### Flashing the disk image

Flash the disk image and cloud-init files onto SD cards with the [Hypriot `flash` tool](https://github.com/hypriot/flash).

```
$ flash -d /dev/disk2 \
  -u nodeX-user-data \
  -F network-config \
  hypriotos-rpi-v1.12.2.img
```

### First boot

Boot each node into Hypriot and let it resize the file system and apply `user-config`. Once a console prompt is available, verify that `wlan0` is up, then update and install dependencies.

```
$ sudo apt-get update
$ sudo apt-get upgrade -y
$ sudo apt-get install -y kubectl kubeadm kubelet
```

Restart.

```
$ sudo init 6
```

Verify that `wlan0` and `eth0` both came up. Verify that you can ping other nodes.

### Initialize the cluster with kubeadm

Log into the master node and [install Kubernetes with kubeadm according to the documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/). I configured the master to advertise the API server on the `eth0` interface. Otherwise nothing custom.

```
$ sudo kubeadm init --apiserver-advertise-address=10.1.1.1
...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.1.1.1:6443 --token TOKEN_STRING \
    --discovery-token-ca-cert-hash sha256:LONG_HASH_VALUE
```

After a few minutes, the control plane should be installed. Copy the `kubeadm join` command from the output for use later.

Copy the Kubeconfig file to your home directory as shown in the output and verify `kubectl` works with it.

```
$ sudo cp /etc/kubernetes/admin.conf /home/k8s/.kube/config
$ sudo chown k8s:k8s /home/k8s/.kube/config
$ kubectl get nodes
```

### Install the overlay network

Running `kubectl get nodes` should show the cluster master node is in `NotReady` state. To finish installation, install a pod overlay network. I used [Weave Net](https://github.com/weaveworks/weave).

> The Kubernetes documentation Documentation currently states that Calico is the only overlay network used for e2e testing, but the Calico image is not compiled for armv7.

Download and apply the Weave Net manifest to install the overlay network.

```
$ curl -L -o weave.yaml https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')
$ kubectl apply -f weave.yaml
```

Watch the overlay pods until they enter the "Running" state.

```
$ watch kubectl -n kube-system get pods
```

The master should now be ready.

```
$ kubectl get nodes
```

### Adding worker nodes

On each worker node, run the `kubeadm join` copied from `kubeadm init` output.

```
# Example
$ sudo kubeadm join 10.1.1.1:6443 --token TOKEN_STRING \
>     --discovery-token-ca-cert-hash sha256:LONG_HASH_VALUE
```

Watch the status of the nodes on the cluster master.

```
$ watch kubectl get nodes
```

Once all the nodes are in the "Ready" state, you should have a working cluster.

### Verify Kubernetes

As a simple test of Kubernetes, run an Nginx pod (example from [this article by Eduard Iskandarov](https://itnext.io/building-a-kubernetes-cluster-on-raspberry-pi-and-low-end-equipment-part-1-a768359fbba3)). This is simpler than writing up and applying a manifest.

```
$ kubectl create deployment nginx --image=nginx
$ kubectl create service nodeport nginx --tcp=80:80
```

Once the pod is ready, use `kubectl describe service` to get the port number.

```
$ kubectl describe service nginx
Name:                     nginx
Namespace:                default
Labels:                   app=nginx
Annotations:              <none>
Selector:                 app=nginx
Type:                     NodePort
IP:                       10.110.64.240
Port:                     80-80  80/TCP
TargetPort:               80/TCP
NodePort:                 80-80  32474/TCP
Endpoints:                10.44.0.1:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

You should now be able to send an HTTP request to that port on any node and get a response.

```
# Note: This is from a node *outside* the cluster, so uses the wlan0 address
$ curl http://192.168.4.42:32474/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

This should work for all nodes, including the master.

Congratulations, you now have your own Kubernetes cluster!
