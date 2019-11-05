# Kubernetes The Fun Way

#### Sign up for the Kubernetes The Fun Way Rancher Master Class: https://info.rancher.com/kubernetes-the-fun-way-online-training

This tutorial was greatly inspired by [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) in order to bootstrap Kubernetes (K3s) on a [Pine64 Clusterboard](https://www.pine64.org/clusterboard/).

The purpose is to build a full-featured home cluster with relatively affordable hardware.

![Cluster](images/cluster.gif)

> Please note, that just because I managed to get it working in a somewhat stable manner, it is not production ready by far! (Running ceph on USB flash drives is not a smart idea)

## Target Audience

The target audience for this tutorial is someone who has completed [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) and would like to understand the challenges of running Kubernetes on bare-metal hardware. General knowledge regarding Arm64 will prove to be extremely useful.

## Cluster Details

- [Armbian](https://www.armbian.com/sopine-a64/) v5.83 (with a custom kernel)
- [Kubernetes (K3s)](https://github.com/rancher/k3s) v1.14.1 on K3s v0.5.0
- [Rook-ceph](https://github.com/rook/rook) v1.0.2
- [Argo Tunnel](https://github.com/cloudflare/cloudflare-ingress-controller) v0.6.5

## Hardware

- [1x Pine64 Clusterboard](https://www.pine64.org/clusterboard/) what made this project possible
- [7x SoPine A64-LTS](https://www.pine64.org/sopine/) amazing arm64 compute modules with 2GB Ram
- [7x SanDisk 32Gb MicroSD cards](https://www.bol.com/nl/p/sandisk-ultra-micro-sdhc-32gb-uhs1-a1-met-adapter/9200000080737253/) boot drives
- [8x SanDisk 64Gb USB flash drives](https://www.mediamarkt.nl/nl/product/_sandisk-cruzer-ultra-usb-3-0-64-gb-1262970.html) 7 were used for ceph storage, one for dissasmbly test
- [Pine64 Cluster Case](https://www.c4labs.com/product/presale-pine64-cluster-case-pine64-clusterboard-with-7-sopine-compute-module-slots/) the only available case for this cluster

## Steps

This tutorial is intended for Arm64 compatible Single Board Computers, any combination of them should work, but it will require you to build a custom Kernel image.

- [Hardware Setup](docs/01-hardware-setup.md)
- [Installing Kubernetes](docs/02-installing-kubernetes.md)
- [Creature Comfort](docs/03-creature-comfort.md)
- [Rook-ceph](docs/04-rook-ceph.md)
- [Database setup](docs/05-database-setup.md)
- [Wordpress](docs/06-wordpress.md)
