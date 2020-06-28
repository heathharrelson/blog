---
title: Testing the Kubernetes NFS Client Provisioner
description: Findings from testing the Kubernetes NFS client provisioner for persistent volumes.
date: 2020-06-27T16:20:41.745-07:00
draft: false
tags:
- kubernetes
- raspberry pi
- storage
---

While Kubernetes pods are ephemeral, most applications have _some_ state that we want to survive a restart, which means dealing with Kubernetes' [persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).<!--more-->For the bare-metal clusters I deal with at work, we manually create a subdirectory on an NFS share, then configure the relevant containers to mount the directory as a volume. Setting this up manually doesn't feel particularly "cloud native," so I was intrigued when I came across this [opensource.com article on creating dynamic persistent volumes using the NFS client provisioner](https://opensource.com/article/20/6/kubernetes-nfs-client-provisioning).

The use case for the NFS client provisioner is essentially what I described above. If you have an existing NFS share, the NFS client provisioner can automatically create subdirectories on it to satisfy persistent volume claims. Sounds great! Let's, try it, shall we?

Following the steps from the [opensource.com article](https://opensource.com/article/20/6/kubernetes-nfs-client-provisioning), I cloned the [external storage respository from GitHub](https://github.com/kubernetes-incubator/external-storage) to my Raspberry Pi cluster.

```
HypriotOS/armv7: k8s@k8smaster in ~
$ git clone https://github.com/kubernetes-incubator/external-storage.git
```

I applied the access control permissions.

```
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ kubectl apply -f rbac.yaml
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
```

I configured [`deployment-arm.yaml`](https://github.com/kubernetes-incubator/external-storage/blob/master/nfs-client/deploy/deployment-arm.yaml) for my NFS share, then deployed the NFS client provisioner by applying the manifest.

```
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ kubectl apply -f deployment-arm.yaml
deployment.apps/nfs-client-provisioner created
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-cbb9b57db-rqrz6   1/1     Running   0          10m
nginx-f89759699-mz2tg                    1/1     Running   0          11d
```

Finally, I created the [storage class](https://github.com/kubernetes-incubator/external-storage/blob/master/nfs-client/deploy/class.yaml), the [volume claim](https://github.com/kubernetes-incubator/external-storage/blob/master/nfs-client/deploy/test-claim.yaml), and the [test pod](https://github.com/kubernetes-incubator/external-storage/blob/master/nfs-client/deploy/test-pod.yaml) by applying the related manifests.

```
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ kubectl apply -f class.yaml
storageclass.storage.k8s.io/managed-nfs-storage created
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ kubectl get storageclass
NAME                  PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   nfs-storage   Delete          Immediate           false                  25s
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ kubectl apply -f test-claim.yaml
persistentvolumeclaim/test-claim created
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ kubectl apply -f test-pod.yaml
pod/test-pod created
```

As expected, the NFS client provisioner created a directory for the persistent volume claim:

```
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ ls -l /cluster-data
total 24
drwxrwxrwx 2 nobody nogroup  4096 Jun 27 17:15 default-test-claim-pvc-7a0cafd4-5958-4671-aa59-6e5a9b1086e7
drwx------ 2 root   root    16384 Jun 17 18:19 lost+found
drwxr-xr-x 3 k8s    k8s      4096 Jun 21 18:37 manifests
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ ls -l /cluster-data/default-test-claim-pvc-7a0cafd4-5958-4671-aa59-6e5a9b1086e7/
total 0
-rw-r--r-- 1 nobody nogroup 0 Jun 27 17:15 SUCCESS
```

So it definitely works. However, there are two things that concern me, one minor and one major.

The minor issue is that if you aren't careful with configuration, your data will be deleted when you delete the volume claim.

```
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ ls /cluster-data
default-test-claim-pvc-7a0cafd4-5958-4671-aa59-6e5a9b1086e7  lost+found  manifests
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ kubectl delete pvc test-claim
persistentvolumeclaim "test-claim" deleted
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ ls /cluster-data
lost+found  manifests
```

This probably isn't a major problem, as long as you're aware of it. You can also configure the storage class to archive the data when the PVC is deleted.

```
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ git diff class.yaml
diff --git a/nfs-client/deploy/class.yaml b/nfs-client/deploy/class.yaml
index 4d3b4805..ba774236 100644
--- a/nfs-client/deploy/class.yaml
+++ b/nfs-client/deploy/class.yaml
@@ -2,6 +2,6 @@ apiVersion: storage.k8s.io/v1
 kind: StorageClass
 metadata:
   name: managed-nfs-storage
-provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'
+provisioner: nfs-storage # or choose another name, must match deployment's env PROVISIONER_NAME'
 parameters:
-  archiveOnDelete: "false"
+  archiveOnDelete: "true"
$ kubectl delete pvc test-claim
persistentvolumeclaim "test-claim" deleted
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ ls /cluster-data
archived-default-test-claim-pvc-734bb4eb-e070-48dc-9071-a56c9ef71c6d  manifests
lost+found
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ ls /cluster-data/archived-default-test-claim-pvc-734bb4eb-e070-48dc-9071-a56c9ef71c6d/
SUCCESS
```

The more concerning issue is that the NFS client provisioner creates directories that are world writable (😱😱😱).

```
HypriotOS/armv7: k8s@k8smaster in ~/external-storage/nfs-client/deploy
$ ls -l /cluster-data
total 24
# NOOOOOOOOOOOOOOO!
drwxrwxrwx 2 nobody nogroup  4096 Jun 27 20:49 default-test-claim-pvc-f8129e20-d910-42b2-9d01-78464fa6aee0
drwx------ 2 root   root    16384 Jun 17 18:19 lost+found
drwxr-xr-x 3 k8s    k8s      4096 Jun 21 18:37 manifests
```

Even worse, the [directory permissions are hardcoded](https://github.com/kubernetes-incubator/external-storage/blob/master/nfs-client/cmd/nfs-client-provisioner/provisioner.go#L69) in the source.

```go
func (p *nfsProvisioner) Provision(options controller.VolumeOptions) (*v1.PersistentVolume, error) {
  // ...

	fullPath := filepath.Join(mountPath, pvName)
	glog.V(4).Infof("creating path %s", fullPath)
	if err := os.MkdirAll(fullPath, 0777); err != nil {
		return nil, errors.New("unable to create directory to provision new pv: " + err.Error())
	}
	os.Chmod(fullPath, 0777)

	// ...
}
```

It's possible that I'm missing something, but this seems not great. I could definitely update the configuration to run as an unprivileged user and create directories with safer permissions, but I prefer not to maintain this kind of low-level cluster infrastructure. I suspect we'll stick with manual volume configuration, at least until our IT organization offers a more dynamic storage solution.

## Links

* [Opensource.com: Provision Kubernetes NFS clients on a Raspberry Pi homelab](https://opensource.com/article/20/6/kubernetes-nfs-client-provisioning)
* [GitHub: Kubernetes incubator external-storage repo](https://github.com/kubernetes-incubator/external-storage)
