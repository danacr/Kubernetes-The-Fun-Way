# Creature Comfort

## Local kubectl, helm, ingress and, the dashboard

There are certain things one expects from a Kubernetes cluster, but they are not a given on an Arm64 K3s cluster behind multiple NATs. Let's fix that.

To enable remote access (only works on the same LAN), export your kubeconfig from the master node.

```
k3s kubectl config view --raw >clusterconfig.yaml
```

Grab it on your local machine using SCP: `scp root@192.168.178.100:clusterconfig.yaml /home/dan/`

Add the cluster, context and user from `clusterconfig.yaml`to your local kubeconfig, making sure you change the cluster ip from `localhost` to the one of the master node.

![local](../images/local.png)

Since we are still running helm which requires tiller, it needs to be installed on the cluster.

Create the tiller service account and the role bindings using:

```
kubectl create -f yamls/arm-tiller.yaml
```

Because tiller does not offer an Arm64 image, we have to use [Jesse Stuart's](https://github.com/jessestuart/tiller-multiarch) instead:

```
helm init --service-account tiller --tiller-image=jessestuart/tiller
```

In my current setup, I cannot assign a public IP address for the cluster, but I still want to be able to expose services. [CloudFlare's Argo Tunnel](https://developers.cloudflare.com/argo-tunnel/quickstart/) offers an interesting [kubernetes ingress controller](https://github.com/cloudflare/cloudflare-ingress-controller) that relies on tunnels.

> Note, this is a paid service, free alternatives are available, such as [Alex Ellis's Inlets](https://github.com/alexellis/inlets). I chose Argo because it was faster to set up.

First, you need to install cloudflared: https://developers.cloudflare.com/argo-tunnel/downloads/ (e.g. `brew install cloudflare/cloudflare/cloudflared`)

```
cloudflared tunnel login
helm repo add cloudflare https://cloudflare.github.io/helm-charts
helm repo update
kubectl create secret generic mad.md --from-file="$HOME/.cloudflared/cert.pem"
```

Replace mad.md with your domain. I also rebuilt the argo tunnel image to support Arm64 so you don't have to:

```
helm install --name argo --namespace default \
    --set rbac.create=true \
    --set controller.ingressClass=argo-tunnel \
    --set controller.logLevel=6 \
    --set controller.image.repository=danacr/argo-tunnel-arm \
    --set controller.image.tag=latest \
    cloudflare/argo-tunnel
```

I thought it would be nice to use the kubernetes dashboard in this project as it provides an easy way to see the cluster status. I recommend using a proper metrics and log stack (Prometheus/Graphana/ELK) if you are running production workloads.

```
kubectl apply -f yamls/arm-dashboard.yaml
```

Get the service account token using:
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kubernetes-dashboard | awk '{print $1}')
```

You can reach the dashboard using `kubectl proxy` and navigating to:
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default

Next: [Rook-ceph](04-rook-ceph.md)
