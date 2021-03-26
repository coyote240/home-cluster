# Rancher k3s Pi Cluster

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
