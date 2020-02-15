# Rook-ceph

## Creating a dynamically provisioned, replicated storage pool using USB flash drives

I could have chosen NFS, or using hostDir for volumes, but that did not seem good enough. Since ceph has arm64 support, it made sense to give it a try.

Let's start with formatting the drives:

> It is recommended to run these commands on all nodes using tmux (From step 2: [Installing Kubernetes](02-installing-kubernetes.md))

Find the /dev of the disk drives: `fdisk -l | grep '^Disk'`

![fdisk](../images/fdisk.png)

Since the USB drives are available at /dev/sda, we will format them to ext4, create a directory where we intend to mount them and eventually mount them.

```
mkfs.ext4 /dev/sda1
mkdir /media/storage
mount /dev/sda1 /media/storage
df -H | less
```

![df](../images/df.png)

To make the mounts survive reboots, fstab has to be edited. This is still possible to do while all the windows are synchronized `vim /etc/fstab`

Insert this line at the end:

```
/dev/sda1               /media/storage           ext4    defaults        1 2
```

Now the storage will survive reboots. Let's get to the ceph installation.

```
helm tiller run helm repo add rook-release https://charts.rook.io/release
helm tiller run helm install  --namespace rook-ceph rook-release/rook-ceph --name rook-ceph --version v1.0.6
```

> Note, v1.0.6 is the last working version for our setup. This is what happens if you install post 1.10:

```
k logs -n rook-ceph csi-cephfsplugin-provisioner-68b89bfd6c-9c22n csi-cephfsplugin
standard_init_linux.go:207: exec user process caused "exec format error"
```

Some of the new components don't play well with arm64

The next step is to install the [Ceph Cluster CRD](https://github.com/rook/rook/blob/master/Documentation/ceph-cluster-crd.md#ceph-cluster-crd).

```

kubectl create -f yamls/cephcluster.yaml

```

I recommend waiting till all the pods are in a completed or Running state and then creating the [ceph block pool](https://github.com/rook/rook/blob/master/Documentation/ceph-block.md):

```

kubectl apply -f yamls/blockstorage.yaml

```

Once that is setup, it makes sense to tag our ceph block pool as the default StorageClass:

```

kubectl patch storageclass rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

```

> Note, some of these steps might take a while to complete. Make sure to check kubectl get pods -A to see the progress.

Ceph comes with a [built-in dashboard](https://github.com/rook/rook/blob/master/Documentation/ceph-dashboard.md), and it seemed like a good idea to have it deployed, you can grab the password using:

```

kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo

```

Create a port-forward using:

```

kubectl port-forward -n rook-ceph svc/rook-ceph-mgr-dashboard 8443

```

Username is `admin`, the dashboard will be available at: https://localhost:8443/#/cephfs

> If you would like to know more about the status of your storage pool, the [Rook Toolbox](https://github.com/rook/rook/blob/master/Documentation/ceph-toolbox.md) worked out great for me.

![ceph](../images/ceph.png)

Next: [Database setup](05-database-setup.md)
