# Creature Comfort

## Local kubectl, helm, ingress and, the dashboard

There are certain things one expects from a Kubernetes cluster, but they are not a given on an Arm64 K3s cluster behind multiple NATs. Let's fix that.

To enable remote access (only works on the same LAN), export your kubeconfig from the master node.

```
k3s kubectl config view --raw >clusterconfig.yaml
```

Grab it on your local machine using SCP: `scp root@192.168.178.100:clusterconfig.yaml /home/dan/`

Add the cluster, context and user from `clusterconfig.yaml`to your local kubeconfig, making sure you change the cluster ip from `localhost` to the one of the master node.

```
wget https://github.com/danacr/drone-cloudflared/releases/download/v0.2/cloudflared-arm64.tar.gz
tar xf cloudflared-arm64.tar.gz -C /usr/local/bin/
cloudflared tunnel login
```

Cloudflare will refuse to connect to https://localhost:6443 if you don't add the server-ca certificate to your trusted certificates:

```
cp /var/lib/rancher/k3s/server/tls/server-ca.crt /usr/local/share/ca-certificates/kubernetes.crt
update-ca-certificates
```

Now we can add the right config file for the cloudflare runnel

```
cp yamls/config.yml /etc/cloudflared/ #make sure to change the hostname to your preferred subdomain on cloudflare
cp .cloudflared/cert.pem /etc/cloudflared/
cloudflared service install
```

If everything went well, we have to add an entry to out local `.kube/config` which will allows us to connect to our cluster. I think it is easier that I show you my config instead of just explaining this part:

```yaml
apiVersion: v1
clusters:
  - cluster:
      certificate-authority-data: "the certificate we extracted using k3s kubectl config view --raw >clusterconfig.yaml"
      server: https://192.168.86.100:6443 # the ip address of your master node on your local network
    name: k3s # I use this as the local network config
  - cluster:
      insecure-skip-tls-verify: false # this is required as cloudflare overwrites our certificate when it is proxying
      server: https://k3s.mad.md # the hostname you specified in the config.yml for cloudflare
    name: k3s-remote # this one uses cloudlfared to connect back to the cluster
contexts:
  - context:
      cluster: k3s
      user: k3s
    name: k3s
  - context:
      cluster: k3s-remote
      user: k3s # we are using the same user as the local connection, so there is no need to specify it twice
    name: k3s-remote
current-context: k3s
kind: Config
preferences: {}
users:
  - name: k3s
    user:
      password: "password created by k3s"
      username: admin
```

Congratulations! Now you should be able to access your cluster remotely.

> Note, kubectl will work remotely, but helm won't work (it has to do something with helm using another port, but I am not certain, happy to hear if someone figures it out)

![local](../images/local.png)
We will run [tillerless helm](https://github.com/rimusz/helm-tiller), otherwise we have to install an arm-based tiller in our cluster.

```
helm init --client-only
helm plugin install https://github.com/rimusz/helm-tiller
helm tiller start
```

> Note, this will require to apped `helm tiller run` to all your regular commads: `helm ls` becomes `helm tiller run helm ls`. It is a small price to pay for not having to install tiller on your cluster

In my current setup, I cannot assign a public IP address for the cluster (It is running behind multiple NATs), but I still want to be able to expose services. [CloudFlare's Argo Tunnel](https://developers.cloudflare.com/argo-tunnel/quickstart/) offers an interesting [kubernetes ingress controller](https://github.com/cloudflare/cloudflare-ingress-controller) that relies on tunnels.

> Note, this is a paid service, free alternatives are available, such as [Alex Ellis's Inlets](https://github.com/alexellis/inlets). I chose Argo because it was faster to set up.

First, you need to install cloudflared: https://developers.cloudflare.com/argo-tunnel/downloads/ (e.g. `brew install cloudflare/cloudflare/cloudflared`)

```
cloudflared tunnel login
helm tiller run helm repo add cloudflare https://cloudflare.github.io/helm-charts
helm tiller run helm repo update
kubectl create secret generic mad.md --from-file="$HOME/.cloudflared/cert.pem"
```

Replace mad.md with your domain. I also rebuilt the argo tunnel image to support Arm64 so you don't have to:

```
helm tiller run helm install --name argo --namespace default \
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

You can reach the dashboard using

```
kubectl port-forward -n kube-system svc/kubernetes-dashboard 8443:443
```

and navigating to `https://localhost:8443`

Next: [Rook-ceph](04-rook-ceph.md)
