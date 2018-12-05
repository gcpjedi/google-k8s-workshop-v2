# Securing Network and Container Runtime

Module objectives

1. Create and test a network security policy
1. Create and test a pod security policy
1. Enable Istio mutual TLS
1. Use Istio Whitelist/Blacklist
1. Setup a GKE Private Cluster, make sure nodes don't have public IPs
1. Use Master Authorized Networks to secure access to the cluster master endpoint
1. Enable Metadata Concealment to prevent pods from accessing certain VM metadata
1. Enable and test Cloud IAP for the cluster

## Setup a GKE Private Cluster

GKE cluster endpoints by default have public IPs. That mean users outside may access both master API and SSH to the nodes.

GKE private cluster is a method to deploy cluster with private IPs only and restrict outside access.  The method uses VPC peering to connect master with nodes. Nodes have no Internet access. To access GCP service like Container Registry they use Private Google Access.

To interact with the cluster one need to provision a vm inside the cluster network or setup master authorized networks.

1. Create a private kubernetes cluster on GKE

    ```shell
    gcloud container clusters create gke-workshop-1 \
        --create-subnetwork name=workshop-subnet \
        --enable-master-authorized-networks \
        --enable-ip-alias \
        --enable-private-nodes \
        --enable-private-endpoint \
        --master-ipv4-cidr 172.16.0.32/28 \
        --no-enable-basic-auth \
        --no-issue-client-certificate \
        --cluster-version 1.11.3 \
        --num-nodes 1 \
        --machine-type n1-standard-2
    ```

    `--enable-private-nodes` configures Nodes to use internal IP addresses

    `--enable-private-endpoint` disables public master API endpoint

    `--enable-ip-alias` is required to enable VPC peering between master and nodes

    `--enable-master-authorized-networks` configure cluster to allow connections from other subnets then one the cluster itself uses

1. Try to connect to the clsuter from your laptop.

    `kubectl` hangs as private endpoint is not available over the internet

1. Create jumpbox in the same subnet as Kuberentes cluster

    ```shell
    # create jumpbox instance in the same subnet
    gcloud compute instances create jumpbox-inner \
      --subnet=workshop-subnet \
      --machine-type=f1-micro \
      --scopes=cloud-platform

    gcloud compute ssh jumpbox-inner
    ```

    `--scopes=cloud-platform` option allows `gcloud` SDK download cluster credentials from within the instance

    `--subnet=workshop-subnet` creates machine is the same subnet as cluster

    Jumpbox has both public and private IPs. You SSH to the jumpbox with its public interface and connect to the Kubernetes cluster using private IP.

1. Verify that you can access Kubernetes from the jumpbox

    ```shell
    $ gcloud container clusters get-credentials gke-workshop-1 --zone=europe-west1-d
    Fetching cluster endpoint and auth data.
    kubeconfig entry generated for gke-workshop-1.

    $ sudo apt-get update && sudo apt-get install kubectl

    $ kubectl get nodes -o wide
    NAME                                            STATUS   ROLES    AGE   VERSION          INTERNAL-IP   EXTERNAL-IP
      OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
    gke-gke-workshop-1-default-pool-f80a37f2-wxfj   Ready    <none>   16m   v1.11.3-gke.18   10.70.0.2 Container-Optimized OS from Google   4.14.65+         docker://17.3.2

    $ kubectl cluster-info
    Kubernetes master is running at https://172.16.0.34
    GLBCDefaultBackend is running at https://172.16.0.34/api/v1/namespaces/kube-system/services/default-http-backend:ht
    tp/proxy
    Heapster is running at https://172.16.0.34/api/v1/namespaces/kube-system/services/heapster/proxy
    KubeDNS is running at https://172.16.0.34/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    Metrics-server is running at https://172.16.0.34/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
    ```

1. Now create the vm outside the cluster subnet

    ```shell
    $ gcloud compute instances create jumpbox-outer \
      --machine-type=f1-micro \
      --scopes=cloud-platform
    Created [https://www.googleapis.com/compute/v1/projects/project-aleksey-zalesov/zones/europe-west1-d/instances/jumpbox-outer].
    NAME           ZONE            MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
    jumpbox-outer  europe-west1-d  f1-micro                   10.132.0.2   35.205.132.74  RUNNING

    $ gcloud compute ssh jumpbox-outer
    ```

1. Can you connect to the Kubernetes cluster from `jumpbox-outer`? Try it.

1. To enable connections from the vm outside cluster subnet add the machines private IP address to the list of masters authorized networks.

    ```shell
    gcloud container clusters update gke-workshop-1 \
        --enable-master-authorized-networks \
        --master-authorized-networks 10.132.0.2/32
    ```

    This time you should be able to access Kubernetes API from `jumpbox-outer`

## Clean up

Delete the provisioned resources

```shell
gcloud compute instances delete jumpbox-inner jumpbox-outer

gcloud container clusters delete gke-workshop-1
```
