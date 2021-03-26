# Rancher k3s Pi Cluster

The purpose of this project is to learn more about using [k3s](https://k3s.io)
Kubernetes distribution in an edge environment, specifically on a Raspberry Pi
v4, provided by my company for Pi-Day 2021.

My goals for the project are to:

* Install and run k3s on the pi
* Experiment with automatic deployment of manifests
* Experiment with adding a storage provider via a network NFS share
* Install and configure Tekton and other tools via automatic deployment
* Assemble a build pipeline for compiling Rust binaries for ARM

My project today consists of a single k3s node. I have questions I would like to
answer that may or may not be done today:

* Can k3s clusters be run across processor architectures?
* Will k3s schedule workloads across nodes if the processor architecture
  differs?
* If so, does k3s manage multiple versions of specific containers when multiple
  architectures are included?

## Installation

On the pi:

```shell
curl -sfL https://get.k3s.io | sh -
```

## Access

On the pi:

```shell
sudo cp /etc/rancher/k3s/k3s.yaml .
sudo chown $USER:$USER k3s.yaml
chmod 600 k3s.yaml
```

Now edit the k3s.yaml file to replace all references to `default` with a name of
your choosing. I chose `rancher`. Change the value of `server` to your pi's IP
or domain name.

On your local machine:

```shell
scp $HOST:~/k3s.yaml ~/.kube/k3s.yaml
export KUBECONFIG=$HOME/.kube/k3s.yaml:$KUBECONFIG
```

To test:

```shell
kubectl config show
```

Observe your cluster added to your list of already configured clusters (if
you have any).

```shell
kubectl config use-context rancher
kubectl get namespaces
```

## Storage

My cluster will require storage. k3s is configured by default to use local
storage for volume claims, but I want my cluster to use an NFS share on my home
NAS to provide storage instead. To do this, I've written an NFS volume claim,
found at nfs-claim.yaml. This specifies storage capacity of 200Gi, and
references the NFS share I've set aside on my NAS for the purpose

## Automatic Deployment

A pretty cool feature of k3s is its ability to automatically deploy manifests.
As I'm most interested in using k3s at the edge as opposed to in a cloud
environment, this is great for bringing up devices like pis automatically. Any
manifest that is copied to `/var/lib/rancher/k3s/server/manifests` is deployed
automatically as if you'd done a `kubectl apply -f`.

To test this feature out, I copied nfs-provider.yaml to the k3s manifests path,
and right away, I see my persistent volume has been created.

```shell
kubectl get pv

NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nfs-akasha   200Gi      RWO,RWX        Retain           Available           manual                  13s
```

Manifests installed using automatic deployment are managed as add-on custom
resources, and my NFS provider is no exception:

```shell
kubectl get addons -A

NAMESPACE     NAME                        AGE
kube-system   ccm                         85m
kube-system   coredns                     85m
kube-system   local-storage               85m
kube-system   aggregated-metrics-reader   85m
kube-system   auth-delegator              85m
kube-system   auth-reader                 85m
kube-system   metrics-apiservice          85m
kube-system   metrics-server-deployment   85m
kube-system   metrics-server-service      85m
kube-system   resource-reader             85m
kube-system   rolebindings                85m
kube-system   traefik                     85m
kube-system   nfs-provider                44m
```

## Install Tekton

Learning what we have about automatic installation, this makes installing Tekton
Pipelines a quick one-liner:

```shell
sudo wget -O /var/lib/rancher/k3s/server/manifests/tekton-pipeline.yaml https://github.com/tektoncd/pipeline/releases/download/v0.22.0/release.yaml
```

Right away I see `tekton-pipeline` listed as an addon.

```shell
kubectl get addons -A
NAMESPACE     NAME                        AGE
...
kube-system   tekton-pipeline             1s
```

And, the `tekton-pipelines` namespace has been created as I expected.

```shell
kubectl get ns

NAME               STATUS   AGE
default            Active   86m
kube-system        Active   86m
kube-public        Active   86m
kube-node-lease    Active   86m
tekton-pipelines   Active   29s
```
