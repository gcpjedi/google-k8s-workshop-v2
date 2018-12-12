# Configure GKE

## Module objectives

1. Deploy basic GKE cluster
1. Node pools
1. List available GKE versions
1. Upgrades and auto upgrades
1. Manage add ons
1. Enable and test auto repair
1. Enable and test auto-scaling
1. Deploy an IP Alias cluster
---

## Deploy a basic GKE cluster

```
$ gcloud container clusters create gke-workshop-0 
```

It will take ~4 minutes for Google cloud to create a cluster for you.

If you want you can exit from this command (press `Ctrl + C`) The operation will continue to run in the background. You can check the status of the running operation by executing this command 

```
$ gcloud container operations list
NAME                              TYPE            LOCATION    TARGET              STATUS_MESSAGE  STATUS   START_TIME                      END_TIME
operation-1543617863286-a5458b9a  CREATE_CLUSTER  us-west1-b  gke-workshop-0                      RUNNING  2018-11-30T22:44:23.286618213Z
```

Another thing that you can do if you don't want to wait for the command completion is to specify `--async` flag.

After the cluster is ready the corresponding operation will go to `DONE` state and cluster status will be `RUNNING` You can check your cluster status using the following command 

```
$ gcloud container clusters list
NAME            LOCATION    MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
gke-workshop-0  us-west1-b  1.9.7-gke.11    35.233.246.13  n1-standard-1  1.9.7-gke.11  3          RUNNING
```

After the cluster is ready you can get its credentials.

```
$ gcloud container clusters get-credentials gke-workshop-0
```

Now you can use `kubectl` utility to connect to the cluster. For example, let's verify that the cluster is up and running by listing its nodes

```
$ kubectl get nodes
```

This command should display all cluster nodes. In GCP console open 'Compute Engine' -> 'VM instances' to verify that each node has a corresponding VM.


## Node pools

A node pool is a subset of node instances within a cluster that all have the same configuration. You can create a node pool in your cluster with local SSDs, a minimum CPU platform, preemptible VMs, a specific node image, larger instance sizes, or different machine types. To illustrate the point of using node pools let's create a preemptible node pool with a different machine-type.  

Preemptible pools consist from VMs that last a maximum of 24 hours and provide no availability guarantees. Preemptible VMs are priced lower than standard VMs and it would make sense to use such pool for a noncritical workload or in case you have a lot of replicas of all your pods. 

```
$ gcloud container node-pools create new-pool \
  --num-nodes 3 \
  --machine-type n1-standard-2 \
  --preemptible \
  --cluster gke-workshop-0 
```

In another terminal watch the stream of events from the nodes.

```
$ watch kubectl get nodes --watch
```

While the node pool is creating you can open `Compute Engine -> VM instances` and make sure that created nodes are indeed preemptible. (If you click on instance details `Preemptibility` should be set to on)

When the new node pool nodes are in `Ready` state we can try to simulate the steps that Google Kubernetes engine does under the hood when deleting a node pool.

1. List all nodes in the default pool
    ```
    $ kubectl get node | grep default-pool
    gke-gke-workshop-0-default-pool-66a39856-5vxr   Ready     <none>    3h        v1.9.7-gke.11
    gke-gke-workshop-0-default-pool-66a39856-kdmj   Ready     <none>    3h        v1.9.7-gke.11
    gke-gke-workshop-0-default-pool-66a39856-z9pp   Ready     <none>    3h        v1.9.7-gke.11
    ```
1. Cordon the node so no new Pods will be scheduled  
    ```
    $ kubectl cordon gke-gke-workshop-0-default-pool-66a39856-5vxr
    ```

1. Drain the node to evict all the Pods from it
    ```
    $ kubectl drain gke-gke-workshop-0-default-pool-66a39856-5vxr --ignore-daemonsets
    ```

1. Repeat with other nodes from the `default-pool`.

1. Delete the default pool which now has no active Pods running.

    ```
    gcloud container node-pools delete default-pool --cluster gke-workshop
    ```

Now you should see only Nodes from a node-pool `new-pool`.

## List available GKE versions

You can see that cluster you deployed has version `1.9.7-gke.11`. It is pretty old. What versions are available for deployment?

```
$ gcloud container get-server-config 
```

The latest master version at the time of writing was `1.11.3`.

## Upgrades and auto upgrades 

If we are running an old version it would make sense to upgrade. There are 2 types of upgrades in GKE.

- Master upgrades (upgrades the version of kubernetes master components)
- Node upgrades (upgrades worker nodes)

This type of upgrades should be executed separately let's first try to upgrade the master version. Note that master can be upgraded only 1 minor version at a time, so we cant' upgrade directly to `1.11.3`

```
$ gcloud container clusters upgrade gke-workshop-0 --cluster-version=1.10.9 --master --async
```

After you are done with your master upgrade you can upgrade your workers. You can update worker nodes in a single node pool at a time

```
$ gcloud container clusters upgrade gke-workshop-0 --node-pool new-pool --cluster-version=1.10.9 --async
```

Run `kubectl get nodes` to make sure all your nodes are upgraded.

GKE may also upgrade Nodes version to the latest stable one automatically. To do so update the cluster with `--enable-autoupgrade` option. Automatic upgrades occur at regular intervals.

```
$ gcloud container node-pools create outdated \
  --enable-autoupgrade \
  --node-version 1.9.7-gke.11 \
  --num-nodes 1 \
  --cluster gke-workshop-2
```

## Manage add-ons

Add-ons are optional components that extend general Kubernetes functionality. On GKE you may choose from:

- HttpLoadBalancing, 
- HorizontalPodAutoscaling, 
- KubernetesDashboard,
- Istio, 
- NetworkPolicy

Default addons include HttpLoadBalancing and HorizontalPodAutoscaling

Let's update our cluster to user KubernetesDashboard addon.

Find out using help the exact options to specify additional parameters.

```
$ gcloud container clusters update gke-workshop-0 --update-addons KubernetesDashboard=ENABLED
```

To access the dashboard first run 
```
kubectl proxy
```
Then open web preview on port `8001` and change url path to `/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

Note, that on GKE kubernetes dashboard is deprecated, you can use GKE interface instead.


## Enable and test auto repair

When the auto-repair feature is on, GKE will check the health of your nodes periodically (approximately every 10 minutes). If the Node is not in `Ready` state or doesn't report state at all, GKE will re-create the Node.

We will turn the feature on and then delete one vm from the NodePool.

1. Enable auto-repair feature

    ```
    $ gcloud container node-pools update new-pool \
      --cluster gke-workshop-0 \
      --enable-autorepair
    ```

1. Now show the instances in this node pool

    ```
    $ gcloud compute instances list | grep new-pool
    gke-gke-workshop-0-new-pool-8283d08d-fwsj      us-west1-b      n1-standard-2               true         10.138.0.5       35.247.24.89     RUNNING
    gke-gke-workshop-0-new-pool-8283d08d-wjhm      us-west1-b      n1-standard-2               true         10.138.0.6       35.247.117.119   RUNNING
    gke-gke-workshop-0-new-pool-8283d08d-xcnq      us-west1-b      n1-standard-2               true         10.138.0.7       35.247.43.114    RUNNING
    ```
1. Delete one of the instances

    ```
    $ gcloud compute instances delete gke-gke-workshop-0-default-pool-1ffc4f39-6ml3
    ```

1. Wait for some time to see if the node comes back. Preemptible nodes should come back very quickly.

## Enable and test auto-scaling

One of the major benefits of the cloud is its elasticity. You allocate resources only when you need them. GKE supports this model with autoscaler.

1. Enable cluster autoscaling

    ```
    $ gcloud container clusters update gke-workshop-0 \
        --enable-autoscaling \
        --min-nodes 1 \
        --max-nodes 4 
    ```

1. Now create a workload. How about 5 nginx containers each requesting 1 Gb of memory?  (For now, we are not very interested in the content of the deployment file, because we will examine pods and deployment later)

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

   Save the file as `test-deployment.yml` and run the following command

    ```
    $ kubectl apply -f test-deployment.yml
    ```

1. Let's see how this workload is scheduled. 
    ```
    $ kubectl get pod
    NAME                     READY     STATUS    RESTARTS   AGE
    nginx-66df689b7c-ftvpz   1/1       Running   0          5s
    nginx-66df689b7c-klb4b   1/1       Running   0          5s
    nginx-66df689b7c-q866v   1/1       Running   0          5s
    nginx-66df689b7c-wm87p   0/1       Pending   0          5s
    nginx-66df689b7c-zwdn9   1/1       Running   0          5s
    ```
    In my case 4 pods are scheduled to the nodes, and one is unscheduled.

1. List nodes to make sure that additional worker node was created
    ```
     kubectl get node
    NAME                                        STATUS    ROLES     AGE       VERSION
    gke-gke-workshop-0-new-pool-8283d08d-fwsj   Ready     <none>    1h        v1.10.9-gke.7
    gke-gke-workshop-0-new-pool-8283d08d-q6nm   Ready     <none>    2m        v1.10.9-gke.7
    gke-gke-workshop-0-new-pool-8283d08d-wjhm   Ready     <none>    17m       v1.10.9-gke.7
    gke-gke-workshop-0-new-pool-8283d08d-xcnq   Ready     <none>    24m       v1.10.9-gke.7
    ``` 

## Deploy an IP Alias cluster

Let's run the following 2 commands and compare the network ranges that are assigned to kubernetes pods and kubernetes nodes. Here are my results

```
$ gcloud compute instances list | grep new-pool
gke-gke-workshop-0-new-pool-8283d08d-fwsj  us-west1-b      n1-standard-2               true         10.138.0.5       35.247.24.89     RUNNING
gke-gke-workshop-0-new-pool-8283d08d-t1zt  us-west1-b      n1-standard-2               true         10.138.0.2       35.247.117.119   RUNNING
gke-gke-workshop-0-new-pool-8283d08d-wjhm  us-west1-b      n1-standard-2               true         10.138.0.6       35.247.75.206    RUNNING
gke-gke-workshop-0-new-pool-8283d08d-xcnq  us-west1-b      n1-standard-2               true         10.138.0.7       35.247.98.9      RUNNING

$  kubectl describe pod | grep IP
IP:             10.60.3.3
IP:             10.60.4.7
IP:             10.60.5.2
IP:             10.60.6.3
```

As you can see the cluster is using the same range for pods and nodes. This is because in the recent version of GKE [ip alias](https://cloud.google.com/vpc/docs/alias-ip) are enabled by default. This feature allows you to use native GCP network ranges for pod subnet. To illustrate the benefits that IP alias clusters provide to you let's create a new empty VM.

```
$ gcloud compute instances create example-instance-1
```

When the VM is ready ssh to it. 

```
$ gcloud compute ssh example-instance-1
$ ping 10.60.3.3
PING 10.60.3.3 (10.60.3.3) 56(84) bytes of data.
64 bytes from 10.60.3.3: icmp_seq=1 ttl=63 time=8.92 ms
64 bytes from 10.60.3.3: icmp_seq=2 ttl=63 time=0.305 ms
```

As you can see your pods are directly accessible from another instance. 

Now you can delete the example instance

```
$ gcloud compute instances delete example-instance-1
```

## Run GKE cluster

At the end of the exercise please recreate the cluster using these commands. This will ensure that all have the same configuration during the training.

```
$ gcloud container clusters delete gke-workshop-0 --async

$ gcloud container clusters create gke-workshop \
--cluster-version 1.11.3 \
--num-nodes 4 \
--no-enable-legacy-authorization \
--machine-type n1-standard-2 
```

