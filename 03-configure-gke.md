Configure GKE
=============

Module objectives
-----------------

1. Deploy basic GKE cluster
1. List and describe cluster operations
1. List available GKE versions
1. Manage installed add ons
1. Enable and test auto repair
1. Enable and test auto scaling
1. Enable auto upgrade
1. Deploy an IP Alias cluster and examine the difference with the default one
1. Create a new node-pool and drain all pods into it
1. Use preemptible node-pools
1. Add a node pool with accelerator

---

## Deploy basic GKE cluster

```
gcloud container clusters create gke-workshop-0 --zone europe-west1-d
```

It will take ~4 minutes for Google cloud to create a cluster for you.

## List and describe cluster operations

Check what operations are available for the cluster:

```
gcloud container clusters --help
```

You can get detailed help on the specific command using

```
gcloud container clusters create --help
```

## List available GKE versions

You can see that cluster you deployed has version `1.9.7-gke.11`. It is pretty old. What versions are available for deployment?

```
$ gcloud container get-server-config --zone europe-west1-d
```

The latest master version is `1.11.2`.

## Manage installed add ons

Addons are optional components that extend general Kubernetes functionality. On GKE you may choose from:

- HttpLoadBalancing, 
- HorizontalPodAutoscaling, 
- KubernetesDashboard,
- Istio, 
- NetworkPolicy

Let's create a second cluster with `Istio` service mesh and running Kubernetes `1.11.2`.

Find out using help the exact options to specify additional parameters.

```
gcloud container clusters create gke-workshop-1 \
  --zone europe-west1-d \
  ??
```

The first cluster `gke-workshop-0` is not needed anymore - delete it with `gcloud container clusters delete` command.

Solution:

```
gcloud container clusters create gke-workshop-1 \
  --zone europe-west1-d \
  --cluster-version 1.11.2 \
  --addons Istio

gcloud container clusters delete gke-workshop-0 --zone europe-west1-d
```

## Enable and test auto repair

When the auto-repair feature is on, GKE will check the health of your nodes periodically (approzimately every 10 minutes). If the Node is not in `Ready` state or doesn't report state at all, GKE will re-create the Node.

We will turn the feature on and then delete one vm from the NodePool.

Get the name of NodePool in the cluster

```
$ gcloud container node-pools list --cluster gke-workshop-1 --zone europe-west1-d
NAME          MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
default-pool  n1-standard-1  100           1.11.2-gke.18
```

Enable auto-repair feature

```
gcloud container node-pools update default-pool \
  --cluster gke-workshop-1 \
  --zone europe-west1-d \
  --enable-autorepair
```

Now show the instances wunning in this node pool

```
$ gcloud compute instances list
NAME                                           ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
gke-gke-workshop-1-default-pool-1ffc4f39-6ml3  europe-west1-d  n1-standard-1               10.132.0.6   35.241.129.17   RUNNING
gke-gke-workshop-1-default-pool-1ffc4f39-dlvm  europe-west1-d  n1-standard-1               10.132.0.5   35.195.214.183  RUNNING
gke-gke-workshop-1-default-pool-1ffc4f39-mdjz  europe-west1-d  n1-standard-1               10.132.0.7   35.233.64.25    RUNNING
```

Check `gcloud compute instances delete` help to find out exact syntax how to delete the instance. How can you set up default zone for the project so you don't need to type it with each command?

Solution:

```
gcloud config set compute/zone europe-west1-d

gcloud compute instances delete gke-gke-workshop-1-default-pool-1ffc4f39-6ml3
```

Let's wait about 10 minutes to see if the node comes back.