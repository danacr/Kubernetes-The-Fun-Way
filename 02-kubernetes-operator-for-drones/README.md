# Kubernetes Operator for Drones using [Gobot](https://github.com/hybridgroup/gobot) and [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)

#### Kubernetes Operators - Zürich Go Meetup [recording](https://www.youtube.com/watch?v=JPVgxnsvOs0) and [slides](https://docs.google.com/presentation/d/1VVZ1QPbae4Pnqr-sKEO4knL-2mwz-ijWt-AhJGu6EYQ/edit?usp=sharing)

The purpose is to create an easy way of explaining Kubernetes operators. (Hence the drones)

![Drones](images/drones.gif)

> Please note, do not run this on your production drone clusters :)

## Target Audience

The target audience for episode #2 is someone who has previously worked with Kubernetes resources and is familiar with the API. (Some go knowledge will prove useful)

## Cluster Details

- [Kubernetes (K3s)](https://github.com/rancher/k3s) IMHO, K3s is the best choice for running Kubernetes on single-board computers

## Hardware

- [3x Tello Edu drones](https://www.apple.com/ch-fr/shop/product/HMBE2ZM/A/drone-tello-edu-de-ryze-avec-technologie-dji) really cool and versatile drones for development (1 is no longer functioning)
- [3x Rock Pi S](https://shop.allnet.de/detail/index/sArticle/312948) they are the only ones that support running arm64 and are small enough to be attached to Tello drones.
- [3x Adafruit Powerboost 1000](https://www.digitec.ch/en/s1/product/adafruit-powerboost-1000-electronics-modules-6056837) is required to boost the voltage supplied by the drone battery from 3.8V to 5V in order to power the Rock Pi S
- [3x SanDisk 32Gb MicroSD cards](https://www.bol.com/nl/p/sandisk-ultra-micro-sdhc-32gb-uhs1-a1-met-adapter/9200000080737253/) boot drives for the drones
- [1x Kubernetes the Fun Way Cluster](../01-portable-kubernetes-cluster/README.md) from episode #1

## Steps

- [Hardware Setup](docs/01-hardware-setup.md)
- [Drone Pod](docs/02-drone-pod.md)
- [Drone Controller](docs/03-drone-swarm.md)
- [Swarm Controller](docs/04-drone-swarm.md)
