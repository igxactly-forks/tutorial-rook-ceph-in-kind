# Overview
Steps in this tutoral demonstrates a method to enable block storage in a [KinD](https://kind.sigs.k8s.io/) cluster.

This is possible by doing the following:
- creating a qcow2 image file
- using qemu-nbd to attach the qcow2 file as block storage in Docker
- creating an LVM volume group on the network block storage
- using [OpenEBS/lvm-localpv](https://github.com/openebs/lvm-localpv) to provision filesystem and block storage from the volume group

As a result, running [rook-ceph](https://github.com/rook/rook) running in KinD is possible on PVCs.

### ⚠️ Warning ⚠️
USE A VIRTUAL MACHINE WHEN DOING THIS ON LINUX SYSTEMS!

Mistakes with deploying rook-ceph on Linux systems can result in data loss. Simply using a VM helps you avoid breaking your desktop system when doing storage experiments like this.

## Requirements
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) for Mac/Windows
- [KinD](https://kind.sigs.k8s.io/docs/user/quick-start/)

## Walkthrough
The container [auto-nbd](https://github.com/protosam/auto-nbd) will setup network block storage using `qemu-nbd` and create a volume group to use in the cluster.

The auto-nbd workaround was really built for Docker Desktop, in Linux you can simply create a volume group with LVM.
```
# docker run -d --restart unless-stopped --pid=host --privileged -v "qemu-images:/data/qemu-images" -v '/dev:/dev' -e QCOW2_IMG_PATH=/data/qemu-images/developer.qcow2 -e NBD_DEV_PATH=/dev/nbd0 -e QCOW2_IMG_SIZE=60G -e VG_NAME=myvolgrp --name auto-nbd ghcr.io/protosam/auto-nbd
5f7307d4da4dfe36ddcfff62e8e8119a5b9ebd688af0b74f1f1a300e73ebc67d

# docker ps
CONTAINER ID   IMAGE                       COMMAND            CREATED             STATUS          PORTS   NAMES
5f7307d4da4d   ghcr.io/protosam/auto-nbd   "/entrypoint.sh"   7 minutes ago       Up 7 minutes            qemu-nbd
```

Create a cluster with `kind`.
```
# kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

Install lvm-localpv with helm.
```
# helm upgrade --install lvm-localpv lvm-localpv --namespace openebs-lvm-localpv --create-namespace --repo https://openebs.github.io/lvm-localpv --wait
Release "lvm-localpv" does not exist. Installing it now.
NAME: lvm-localpv
NAMESPACE: openebs-lvm-localpv
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The OpenEBS LVM LocalPV has been installed. Check its status by running:
$ kubectl get pods -n openebs-lvm-localpv -l role=openebs-lvm

For more information, visit our Slack at https://openebs.io/community or view
the documentation online at http://docs.openebs.io/.
```

Create a storage class that uses the volume group from nbd.
```
# kubectl apply -f manifests/storageclass.yaml
storageclass.storage.k8s.io/lvm-myvolgrp created

# kubectl get sc
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
lvm-myvolgrp         local.csi.openebs.io    Delete          Immediate              false                  15s
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  16m
```

Apply the example `example-lvm-pv.yaml` manifest to test the provisioner.
```
# kubectl apply -f manifests/example-lvm-pv.yaml
persistentvolumeclaim/nginx-storage created
pod/nginx created

# kubectl get pods,pv,pvc
NAME        READY   STATUS    RESTARTS   AGE
pod/nginx   1/1     Running   0          9s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
persistentvolume/pvc-580e2a13-704d-4f82-8542-f500374910bc   1Gi        RWO            Delete           Bound    default/nginx-storage   lvm-myvolgrp            8s

NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/nginx-storage   Bound    pvc-580e2a13-704d-4f82-8542-f500374910bc   1Gi        RWO            lvm-myvolgrp   9s
```

Taking this a step further, it is possible to deploy a rook-ceph cluster in this kind cluster.

Start by installing rook-ceph.
```
# helm upgrade --install rook-ceph rook-ceph --create-namespace --namespace rook-ceph --repo https://charts.rook.io/release --wait
Release "rook-ceph" does not exist. Installing it now.
NAME: rook-ceph
NAMESPACE: rook-ceph
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Rook Operator has been installed. Check its status by running:
  kubectl --namespace rook-ceph get pods -l "app=rook-ceph-operator"

Visit https://rook.io/docs/rook/latest for instructions on how to create and configure Rook clusters

Important Notes:
- You must customize the 'CephCluster' resource in the sample manifests for your cluster.
- Each CephCluster must be deployed to its own namespace, the samples use `rook-ceph` for the namespace.
- The sample manifests assume you also installed the rook-ceph operator in the `rook-ceph` namespace.
- The helm chart includes all the RBAC required to create a CephCluster CRD in the same namespace.
- Any disk devices you add to the cluster in the 'CephCluster' must be empty (no filesystem and no partitions).
```

Create use `example-cephcluster.yaml` to create a `CephCluster` object.
```
# kubectl apply -f manifests/example-cephcluster.yaml
cephcluster.ceph.rook.io/rook-ceph created

# kubectl -n rook-ceph get cephclusters
NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE         MESSAGE                  HEALTH   EXTERNAL   FSID
rook-ceph   /var/lib/rook     1          14s   Progressing   Detecting Ceph version
```

Wait for the cluster to be setup.
```
# kubectl -n rook-ceph wait cephclusters rook-ceph --for condition=Ready --timeout=900s
cephcluster.ceph.rook.io/rook-ceph condition met

# kubectl -n rook-ceph get cephclusters,pods
NAME                                 DATADIRHOSTPATH   MONCOUNT   AGE     PHASE   MESSAGE                        HEALTH        EXTERNAL   FSID
cephcluster.ceph.rook.io/rook-ceph   /var/lib/rook     1          2m57s   Ready   Cluster created successfully   HEALTH_WARN              1aab11d6-97b1-4c3d-88c9-066c61e7f263

NAME                                                              READY   STATUS      RESTARTS   AGE
pod/csi-cephfsplugin-7t942                                        2/2     Running     0          2m33s
pod/csi-cephfsplugin-provisioner-7df756967f-55vpm                 5/5     Running     0          2m33s
pod/csi-rbdplugin-provisioner-6566d7677c-kgzwq                    5/5     Running     0          2m33s
pod/csi-rbdplugin-zdb5p                                           2/2     Running     0          2m34s
pod/rook-ceph-crashcollector-kind-control-plane-67d95754d-r8k9m   1/1     Running     0          42s
pod/rook-ceph-mgr-a-5874bb57b6-224lg                              1/1     Running     0          70s
pod/rook-ceph-mon-a-7b75b95847-wgfx6                              1/1     Running     0          2m8s
pod/rook-ceph-operator-67748fc8bd-d4j57                           1/1     Running     0          5m7s
pod/rook-ceph-osd-0-559f6bf48d-8bcbt                              1/1     Running     0          42s
pod/rook-ceph-osd-prepare-genesis-data-0nxbmf-9xmw5               0/1     Completed   0          49s
```

Now you can go through the example configurations for [setting up consumable storage](https://rook.io/docs/rook/v1.11/Getting-Started/example-configurations/#setting-up-consumable-storage) in the Rook documentation.

## Cleanup After Deleting Clusters
Deleting dangling logical volumes is necessary if they were not removed in a kind cluster before the cluster was deleted.
```
% docker exec -it auto-nbd bash
root@b843c9fe6d5d:/# vgs
  VG       #PV #LV #SN Attr   VSize   VFree
  myvolgrp   1  10   0 wz--n- <60.00g <50.00g

root@b843c9fe6d5d:/# lvs --noheading
  pvc-4ba801bf-b7f6-4d85-ba69-4a51b52ac76a myvolgrp -wi------- 1.00g
  pvc-5447de1a-b983-4de1-8dfd-ec6f77e52419 myvolgrp -wi------- 1.00g
  pvc-68f79916-1f64-46f2-babe-2aaf74f0ad2b myvolgrp -wi------- 1.00g
  pvc-6ff7eb10-af74-49b5-9927-f06cce4279a1 myvolgrp -wi------- 1.00g
  pvc-a8a5c7f4-5c0a-4c73-bd21-16b3dc50b27b myvolgrp -wi------- 1.00g
  pvc-a9af3318-2170-4099-ae46-94fc415199c4 myvolgrp -wi------- 1.00g
  pvc-fd34d484-6624-457f-b6aa-e5c65db2c27d myvolgrp -wi------- 1.00g


root@b843c9fe6d5d:/# lvs --noheading | awk '{print "lvremove /dev/"$2"/"$1}' | sh
  Logical volume "pvc-4ba801bf-b7f6-4d85-ba69-4a51b52ac76a" successfully removed
  Logical volume "pvc-5447de1a-b983-4de1-8dfd-ec6f77e52419" successfully removed
  Logical volume "pvc-68f79916-1f64-46f2-babe-2aaf74f0ad2b" successfully removed
  Logical volume "pvc-6ff7eb10-af74-49b5-9927-f06cce4279a1" successfully removed
  Logical volume "pvc-a8a5c7f4-5c0a-4c73-bd21-16b3dc50b27b" successfully removed
  Logical volume "pvc-a9af3318-2170-4099-ae46-94fc415199c4" successfully removed
  Logical volume "pvc-fd34d484-6624-457f-b6aa-e5c65db2c27d" successfully removed
```
