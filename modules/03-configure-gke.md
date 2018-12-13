# Configure GKE

## Module Objectives

1. Deploy a basic GKE cluster
1. Node pools
1. List available GKE versions
1. Upgrades and auto upgrades
1. Manage add-ons
1. Enable and test auto-repair
1. Enable and test auto-scaling
1. Deploy an IP Alias cluster
---

## Deploy a Basic GKE Cluster
1. Create a GKE cluster

  ```shell
  gcloud container clusters create gke-workshop-0
  ```

  > Note: you can specify the `--async` flag if you don't want to wait for the command to complete.

  It will take ~4 minutes for Google Cloud to create a cluster for you.

  If you want you can exit from this command (press `Ctrl + C`). The operation will continue to run in the background. You can check the status of the running operation by executing this command:

  ```shell
  gcloud container operations list
  ```

  ```
  NAME                              TYPE            LOCATION    TARGET              STATUS_MESSAGE  STATUS   START_TIME                      END_TIME
  operation-1543617863286-a5458b9a  CREATE_CLUSTER  us-west1-b  gke-workshop-0                      RUNNING  2018-11-30T:44:23.286618213Z
  ```

  Wait for the operation to reach a `DONE` status.

  Check your cluster status using the following command:

  ```shell
  gcloud container clusters list
  ```

  ```
  NAME            LOCATION    MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
  gke-workshop-0  us-west1-b  1.9.7-gke.11    35.233.246.13  n1-standard-1  1.9.7-gke.11  3          RUNNING
  ```
  Wait for a `RUNNING` status.

1. Get the cluster credentials.

  ```shell
  gcloud container clusters get-credentials gke-workshop-0
  ```

  Now you can use the `kubectl` utility to connect to the cluster. For example, let's verify that the cluster is up and running by listing its nodes

  ```shell
  kubectl get nodes
  ```

  This command should display all cluster nodes. In GCP console open 'Compute Engine' -> 'VM instances' to verify that each node has a corresponding VM.

## Node Pools

A node pool is a subset of node instances within a cluster that all have the same configuration. You can create a node pool in your cluster with local SSDs, a minimum CPU platform, preemptible VMs, a specific node image, larger instance sizes, or different machine types. To illustrate the point of using node pools, let's create a preemptible node pool with a different machine-type.

Preemptible pools consist of VMs that last a maximum of 24 hours and provide no availability guarantees. Preemptible VMs are priced lower than standard VMs and it would make sense to use such pools for a noncritical workload or in case you have a lot of replicas of all your pods.

```shell
gcloud container node-pools create new-pool \
  --num-nodes 3 \
  --machine-type n1-standard-2 \
  --preemptible \
  --cluster gke-workshop-0
```

In another terminal watch the stream of events from the nodes.

```shell
kubectl get nodes --watch
```

While the node pool is creating you can open `Compute Engine -> VM instances` and make sure that created nodes are indeed preemptible. If you click on instance details `Preemptibility` should be set to on. It will take a few minutes for the cluster to reconcile

When the new node pool nodes are in `Ready` state we can try to simulate the steps that Google Kubernetes engine does under the hood when deleting a node pool.

1. List all nodes in the default pool

    ```shell
    kubectl get node | grep default-pool
    ```

    ```
    gke-gke-workshop-0-default-pool-66a39856-5vxr   Ready     <none>    3h        v1.9.7-gke.11
    gke-gke-workshop-0-default-pool-66a39856-kdmj   Ready     <none>    3h        v1.9.7-gke.11
    gke-gke-workshop-0-default-pool-66a39856-z9pp   Ready     <none>    3h        v1.9.7-gke.11
    ```

1. For each `default-pool` node, Cordon the node so that new Pods will not be scheduled

    ```shell
    kubectl cordon gke-gke-workshop-0-default-pool-xxxxxxxx-xxxx
    ```

1. For each `default-pool` node, Drain the node to evict all the Pods from it

    ```shell
    kubectl drain gke-gke-workshop-0-default-pool-xxxxxxxx-xxxx --ignore-daemonsets
    ```

1. Delete the default pool which now has no active Pods running.

    ```shell
    gcloud container node-pools delete default-pool --cluster gke-workshop-0
    ```

Now you should see only Nodes from a node-pool `new-pool`.

## List Available GKE Versions

The cluster you deployed has a version. Use this command to view the `Master` and `Node` versions:
```shell
gcloud container clusters describe gke-workshop-0
```
Look for the following keys in the output:
```shell
currentMasterVersion: 1.10.9-gke.5
currentNodeVersion: 1.10.9-gke.5
```

Let us see all the versions that are available for deployment.

```shell
gcloud container get-server-config
```

The latest master version at the time of writing was `1.11.3`.

## Upgrades and Auto Upgrades

If we are running an old version it would make sense to upgrade. There are 2 types of upgrades in GKE.

- Master upgrades (upgrades the version of kubernetes master components)
- Node upgrades (upgrades worker nodes)

These types of upgrades should be executed separately. Let's first try to upgrade the master version.

> Note that master can be upgraded only one minor version at a time

```shell
gcloud container clusters upgrade gke-workshop-0 --cluster-version=1.10.9 --master --async
```

> Note: This may take a few minutes

You can check the status of the upgrade by checking the container operation

```shell
gcloud beta container operations list --filter TYPE=UPGRADE_MASTER
```

After you are done with your master upgrade you can upgrade your workers. You can update worker nodes in a single node pool at a time

```shell
gcloud container clusters upgrade gke-workshop-0 --node-pool new-pool --cluster-version=1.10.9 --async
```

Try the following two methods to check the status of upgrading the nodes, make sure all your nodes are upgraded.

```shell
gcloud beta container operations list --filter TYPE=UPGRADE_NODES
```

> Note: This may take a few minutes

```shell
watch kubectl get nodes
```

GKE may also upgrade Nodes version to the latest stable one automatically. To do so update the cluster with `--enable-autoupgrade` option. Automatic upgrades occur at regular intervals.

```shell
gcloud container node-pools create outdated \
  --enable-autoupgrade \
  --node-version 1.9.7-gke.11 \
  --num-nodes 1 \
  --cluster gke-workshop-0
```

## Manage Add-ons

Add-ons are optional components that extend general Kubernetes functionality. On GKE you may choose from:

- HttpLoadBalancing
- HorizontalPodAutoscaling
- KubernetesDashboard
- Istio
- NetworkPolicy

Default add-ons include HttpLoadBalancing and HorizontalPodAutoscaling

Let's update our cluster to use the KubernetesDashboard add-on.

```shell
gcloud container clusters update gke-workshop-0 --update-addons KubernetesDashboard=ENABLED
```

To access the dashboard:
1.
  ```shell
  kubectl proxy
  ```

1. Open web preview on port `8001` and change url path to `/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

> Note, that on GKE Kubernetes dashboard is deprecated, you can use the GKE interface instead.


## Enable and Test Auto Repair

When the auto-repair feature is on, GKE will check the health of your nodes periodically (approximately every 10 minutes). If the Node is not in `Ready` state or doesn't report state at all, GKE will re-create the Node.

We will turn the feature on and then delete one vm from the NodePool.

1. Enable auto-repair feature

    ```shell
    gcloud container node-pools update new-pool \
      --cluster gke-workshop-0 \
      --enable-autorepair
    ```

1. Now show the instances in this node pool

    ```shell
    gcloud compute instances list | grep new-pool
    ```

    ```
    gke-gke-workshop-0-new-pool-8283d08d-fwsj      us-west1-b      n1-standard-2               true         10.138.0.5       35.247.24.89     RUNNING
    gke-gke-workshop-0-new-pool-8283d08d-wjhm      us-west1-b      n1-standard-2               true         10.138.0.6       35.247.117.119   RUNNING
    gke-gke-workshop-0-new-pool-8283d08d-xcnq      us-west1-b      n1-standard-2               true         10.138.0.7       35.247.43.114    RUNNING
    ```

1. Delete one of the instances

    ```shell
    gcloud compute instances delete gke-gke-workshop-0-default-pool-1ffc4f39-6ml3
    ```

1. Wait for some time to see if the node comes back. Preemptible nodes should come back very quickly.

## Enable and Test Auto-Scaling

One of the major benefits of the cloud is its elasticity. You allocate resources only when you need them. GKE supports this model with Autoscaling.

1. Enable cluster Autoscaling

    ```shell
    gcloud container clusters update gke-workshop-0 \
      --enable-autoscaling \
      --node-pool new-pool \
      --min-nodes 1 \
      --max-nodes 4
    ```

1. Now create a workload. How about 5 nginx containers each requesting 2 Gb of memory?  

    (For now, we are not very interested in the content of the deployment file, because we will examine pods and deployment later)

    Save this file as `test-deployment.yaml`

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx
    spec:
      selector:
        matchLabels:
          app: nginx
      replicas: 10
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
                memory: "2Gi"
                cpu: "100m"
            ports:
            - containerPort: 80
    ```

    Run the following command

    ```shell
    kubectl apply -f test-deployment.yaml
    ```

1. Let's see how this workload is scheduled.

    ```shell
    kubectl get pod
    ```

    ```
    NAME                     READY     STATUS    RESTARTS   AGE
    nginx-66df689b7c-ftvpz   1/1       Running   0          5s
    nginx-66df689b7c-klb4b   1/1       Running   0          5s
    nginx-66df689b7c-q866v   1/1       Running   0          5s
    nginx-66df689b7c-wm87p   0/1       Pending   0          5s
    nginx-66df689b7c-zwdn9   1/1       Running   0          5s
    ```

    In my case 4 pods are scheduled to the nodes, and one is unscheduled.

1. List nodes to make sure that additional worker node was created

    ```shell
    kubectl get node
    ```

    ```
    NAME                                        STATUS    ROLES     AGE       VERSION
    gke-gke-workshop-0-new-pool-8283d08d-fwsj   Ready     <none>    1h        v1.10.9-gke.7
    gke-gke-workshop-0-new-pool-8283d08d-q6nm   Ready     <none>    2m        v1.10.9-gke.7
    gke-gke-workshop-0-new-pool-8283d08d-wjhm   Ready     <none>    17m       v1.10.9-gke.7
    gke-gke-workshop-0-new-pool-8283d08d-xcnq   Ready     <none>    24m       v1.10.9-gke.7
    ```

## Deploy an IP Alias Cluster

Let's run the following 2 commands and compare the network ranges that are assigned to kubernetes pods and kubernetes nodes. Here are my results

```shell
gcloud compute instances list | grep new-pool
```

```
gke-gke-workshop-0-new-pool-8283d08d-fwsj  us-west1-b      n1-standard-2               true         10.138.0.5       35.247.24.89     RUNNING
gke-gke-workshop-0-new-pool-8283d08d-t1zt  us-west1-b      n1-standard-2               true         10.138.0.2       35.247.117.119   RUNNING
gke-gke-workshop-0-new-pool-8283d08d-wjhm  us-west1-b      n1-standard-2               true         10.138.0.6       35.247.75.206    RUNNING
gke-gke-workshop-0-new-pool-8283d08d-xcnq  us-west1-b      n1-standard-2               true         10.138.0.7       35.247.98.9      RUNNING
```

```shell
kubectl describe pod | grep IP
```

```
IP:             10.60.3.3
IP:             10.60.4.7
IP:             10.60.5.2
IP:             10.60.6.3
```

As you can see the cluster is using the same range for pods and nodes. This is because in the recent version of GKE [ip alias](https://cloud.google.com/vpc/docs/alias-ip) are enabled by default. This feature allows you to use native GCP network ranges for pod subnet. To illustrate the benefits that IP alias clusters provide to you let's create a new empty VM.

```shell
gcloud compute instances create example-instance-1
```

When the VM is ready ssh into it.

```shell
gcloud compute ssh example-instance-1
```

```shell
ping 10.60.3.3
```

```
PING 10.60.3.3 (10.60.3.3) 56(84) bytes of data.
64 bytes from 10.60.3.3: icmp_seq=1 ttl=63 time=8.92 ms
64 bytes from 10.60.3.3: icmp_seq=2 ttl=63 time=0.305 ms
```

As you can see your pods are directly accessible from another instance.

Now you can delete the example instance

```shell
gcloud compute instances delete example-instance-1
```

## Run GKE Cluster

At the end of the exercise please recreate the cluster using these commands. This will ensure that all have the same configuration during the training.

```shell
gcloud container clusters delete gke-workshop-0 --async
```

```shell
gcloud container clusters create gke-workshop \
  --cluster-version 1.11.3 \
  --num-nodes 4 \
  --no-enable-legacy-authorization \
  --machine-type n1-standard-2
```

Next: [Pods & Services](04-pods-and-services.md)
