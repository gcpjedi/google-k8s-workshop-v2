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

After the cluster is ready you can get its credentials using the following command.

```
$ gcloud container clusters get-credentials gke-workshop-0
```

Now you can use `kubectl` utility to connect to the cluster. For example, let's verify that the cluster is up and runnig by listing its nodes

```
$ kubectl get nodes
```

This command should display all cluster nodes. In GCP console open 'Compute Engine' -> 'VM instances' to verify that each node has a corresponding VM.

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

## Enable and test auto-scaling

One of the major benefits of the cloud is its elasticity. You allocate resources only when you need them. GKE supports this model with autoscaler.

```
gcloud container clusters create gke-workshop-2 \
--cluster-version 1.11.2 \
--zone europe-west1-d \
--num-nodes 2 \
--enable-autoscaling \
--min-nodes 1 \
--max-nodes 4 \
--labels=project=gke-workshop-2 \
--enable-autorepair
```

```
gcloud container clusters get-credentials gke-workshop-2 
```

Now create a workload. How about 5 nginx containers each requesting 1 Gb of memory?

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.6
        resources:
          requests:
            memory: "1Gi"
            cpu: "100m"
        ports:
        - containerPort: 80
```

Let's see how this workload is scheduled. 4 pods are scheduled to the nodes, and one is unscheduled.

```
kubectl get deploy --watch
```

Go to the Google Cloud console and see how Cluster Autoscaler creates a new node for the cluster. After node is provisioned the remaining pod is started.

Delete `gke-workshop-1` after verifying that GKE restored the deleted node from the previous experiment.

## Enable auto upgrade

GKE may upgrade Nodes version to the latest stable one automatically. To do so create a cluster with `--enable-autoupgrade` option. Automatic upgrades occur at regular intervals.

```
gcloud container node-pools create outdated \
  --enable-autoupgrade \
  --node-version 1.9.7-gke.11 \
  --num-nodes 1 \
  --cluster gke-workshop-2
```

```
kubectl get nodes
```

Now upgrade the node pool manually to the version of the master. Remeber, automatic upgrades happen based on schedule not immediately.

```
gcloud container clusters upgrade gke-workshop \
  --node-pool outdated \
  --cluster-version 1.11.2
```

Verify all nodes are runninhg the latest version.

```
kubectl get nodes
```

## Deploy an IP Alias cluster

## Create a new node-pool and drain all pods into it

```
gcloud container node-pools create new \
  --enable-autoupgrade \
  --node-version 1.11.2-gke.18 \
  --num-nodes 3 \
  --cluster gke-workshop-2
```

In another terminal watch the stream of events from the nodes.

```
$ kubectl get nodes --watch
```

When the new nodepool nodes are in `Ready` state

```
gcloud container node-pools delete outdated --cluster gke-workshop --async
```

Google Kubernetes engine does several operations under the hood to ensure Pods are evicted from the NodePool and running on a new node. Let's see the process step-by-step.

```
# cordon the node so no new Pods will be scheduled
kubectl cordon gke-gke-workshop-default-pool-33b6dfeb-t7n3 --ignore-daemonsets

# drain the node to evict all the Pods from it
k drain gke-gke-workshop-default-pool-33b6dfeb-t7n3 --ignore-daemonsets
```

Repeate with other nodes from the `default-pool`.

Delete the NodePool which has no active Pods running.

```
gcloud container node-pools delete default-pool --cluster gke-workshop
```

Now you should see only Nodes from a node-pool `new`.
