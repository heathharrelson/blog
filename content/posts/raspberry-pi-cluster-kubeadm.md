---
title: "Building a Raspberry Pi Kubernetes Cluster with Kubeadm"
# TODO description
# TODO title
date: 2020-06-16T19:53:03-07:00
draft: true
---

This article describes how to build a personal [Kubernetes](https://kubernetes.io/) cluster using [Raspberry Pi](https://www.raspberrypi.org/) single-board computers and [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/).<!--more--> There is a TODO LINK companion article that describes how to do the same thing using [k3s](https://k3s.io/).

## Materials

### Hardware

Hardware used:

* 3 Raspberry Pi 4 Model B (4 GB)
* 3 LoveRPi PoE hat for Raspberry Pi 4 Model B (compact)
* TP-Link 8 port PoE gigabit switch

I previously built a cluster using Raspberry Pi 3 Model B+ computers. While this will work, the performance is underwhelming because the computers only have 1 GB of RAM, and it doesn't leave many resources left for applications. The Raspberry Pi 4 is much more capable.

[Power over Ethernet](https://en.wikipedia.org/wiki/Power_over_Ethernet) is optional, but it saves on cabling and makes for a much tidier cluster.

### Software

* [Hypriot 1.12.2](https://github.com/hypriot/image-builder-rpi/releases/tag/v1.12.2)
* [Kubernetes 1.18.3](https://github.com/kubernetes/kubernetes/releases/tag/v1.18.3)
* [Weave Net 2.6.5](https://github.com/weaveworks/weave/releases/tag/latest_release)

Using [Hypriot](https://blog.hypriot.com/downloads/) for the operating system saves installation time, because Docker is preinstalled. However, Hypriot is currently built for 32-bit armv7 architecture. The Raspberry Pi 3 and 4 can run 64-bit binaries, and plenty of the Docker containers available only built for arm64. If that matters to you, you might want to choose something different.

## Setup

### Hypriot Config

Hypriot allows configuring the disk images using [cloud-init](https://cloudinit.readthedocs.io/en/20.2/index.html), which reduces the manual steps required to get the cluster up and makes setup much more repeatable.

#### user-data

Following [`user-data` examples from the Hypriot GitHub repo](https://github.com/hypriot/flash/tree/master/sample), I created a [`user-data` file](https://cloudinit.readthedocs.io/en/20.2/topics/examples.html#yaml-examples) for each host that looks like the following Gist:

TODO GIST
<!-- {{< gist heathharrelson 182d19e5adebc7ae206a51140d190ec0 >}} -->

This will

* set the hostname
* change the username from `pirate` to `k8s`
* add an authorized SSH key
* add a static `/etc/hosts` file
* configure the `wlan0` and `eth0` interfaces
* add the Kubernetes Apt repo

#### network-config

I wanted a static IP for the Ethernet interface, so I commented out the portion of `network-config` that sets up DHCP on `eth0`.

TODO GIST

#### Flashing the Disk Images

Flash disk images onto SD cards with the [Hypriot `flash` tool](https://github.com/hypriot/flash).

```
$ flash -d /dev/disk2 \
  -u nodeX-user-data \
  -F network-config \
  hypriotos-rpi-v1.12.2.img
```

### First Boot

Boot each node into Hypriot and let it resize the file system and apply `user-config`. Once a console prompt is available, verify that `wlan0` is up, then update and install dependencies.

```
$ sudo apt-get update
$ sudo apt-get upgrade -y
$ sudo apt-get install -y kubectl kubeadm kubelet
```

> TODO: Figure out if the following is needed. [GitHub issue mentioned in the Weave Net documentation](https://github.com/weaveworks/weave/issues/3465) has been closed. Update: Yes, `NodePort` service does not work without it. Switching to `iptables-legacy` also made Flannel work.

Switch from `iptables-nf` to `iptables-legacy` to avoid problems with Weave.

```
$ sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
$ sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

Other than the rules used by Docker, Hypriot doesn't set up any firewall rules by default, so no need to open ports.

Restart.

```
$ sudo init 6
```

Verify that `wlan0` and `eth0` both came up. Verify that you can ping other nodes.

### Initialize Cluster With kubeadm

Log into the master node and [install Kubernetes with kubeadm according to the documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/). I configured the master to advertise the API server on the `eth0` interface. Otherwise nothing custom.

```
# Replace 10.1.1.1 with your IP address
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

Copy the Kubeconfig file to home directory as shown in the output and verify `kubectl` works with it.

```
$ sudo cp /etc/kubernetes/admin.conf /home/k8s/.kube/config
$ sudo chown k8s:k8s /home/k8s/.kube/config
```

### Install an Overlay Network

Should show cluster master node is in `NotReady` state. To finish installation, install a pod overlay network. I used Weave Net.

> Aside: Documentation currently states that Calico is the only thing they do e2e testing with, but documentation states it only supports arm64 (Hypriot is still 32 bit). Have never had luck with Flannel.

Downloaded Weave Net manifest.

```
$ curl -L -o weave.yaml https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')
```

Inspected the manifest to see what it was going to do, then applied with `kubectl`

```
$ kubectl apply -f weave.yaml
```

Wait for daemonset pods for overlay to come up.

On each worker node, run the `kubeadm join` copied from `kubeadm init` output.

```
# Example
$ sudo kubeadm join 10.1.1.1:6443 --token TOKEN_STRING \
>     --discovery-token-ca-cert-hash sha256:LONG_HASH_VALUE
```

Check for nodes entering the ready state.

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

Should now be able to send a curl request (or use browser) to that port on any node and get a response.

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

Should work on all nodes, including the master.

## Misc Notes

* Flannel has been removed from the kubeadm docs: https://github.com/kubernetes/website/commit/a9d7ba33446914de7c38adfc6825a5828facaff5#diff-f2a40a553d60c0fe04e84ee9c1bf28a6

## Links

* Hypriot blog: https://blog.hypriot.com/
* Weave Net Documentation: https://www.weave.works/docs/net/latest/kubernetes/kube-addon/
* [Jeff Geerling YouTube series on Kubernetes on Raspberry Pi](https://www.youtube.com/watch?v=kgVz4-SEhbE)
  * [Episode 3 (cluster setup)](https://www.youtube.com/watch?v=N4bfNefjBSw) and [episode 4 (deploying applications)](https://www.youtube.com/watch?v=IafVCHkJbtI) are the most interesting if you aleady know a bit about k8s
  * GitHub repo: https://github.com/geerlingguy/turing-pi-cluster
* NFS client provisioner: https://opensource.com/article/20/6/kubernetes-nfs-client-provisioning


k3s Ansible playbook: https://github.com/rancher/k3s-ansible