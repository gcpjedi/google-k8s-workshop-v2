# Monitoring and Logging

## Module Objectives

1. Enable Stackdriver logging
1. Observe cluster logs
1. Setup an alert

---

## Enable Stackdriver logging

By default, when you create a Kubernetes GKE cluster, the `--enable-cloud-logging` flag is automatically set

Go to the GKE console to check the Kubernetes cluster details: [https://console.cloud.google.com/kubernetes/clusters/details/us-west2-b/gke-workshop](https://console.cloud.google.com/kubernetes/clusters/details/us-west2-b/gke-workshop)

Make sure the "Stackdriver logging" checkbox is turned on

## Observe Cluster Logs

There are two types of logs: `_container_` and `_system_`

  * Container logs are collected from the running containers
  * System logs are produced by cluster components like `kubelet` and `api`
  * There are also events like cluster creation that are produced by Google Cloud

1. Go to the Logging dashboard of the Web Console

1. Select "Convert to advanced filter" to switch to advanced mode

    ![Advanced filter](img/logging-advanced-filter.png)

1. Edit the filter

    ```shell
    resource.type="container"
    resource.labels.namespace_id="default"
    resource.labels.cluster_name="gke-workshop"
    resource.labels.container_name="backend"
    ```

    Click "Submit Filter"

    * You will see logs from the backend container
    * While editing the filter, you can use auto completion ("Control+Space") to explore possible options

1. Find the record that shows the backend container was started and ready to serve requests

    * The line the application prints is "Operating in backend mode..."
    * The message goes to the `textPayload` field
    * Operator `include` is `:`
    * Prepare the filter statement yourself

1. Write a filter for audit operations on the cluster

    ```json
    resource.type=gke_cluster
    resource.labels.cluster_name="gke-workshop"
    protoPayload.@type:AuditLog
    ```

    > Note: This filter will show you when cluster was created

1. Delete the second line. Now you will see operations for all the clusters in the account

1. Some other interesting `resource.type` values to explore are `k8s_cluster`, `k8s_node` and `k8s_pod`

## Setup an Alert

In this exercise you will setup an alert for when new pods start. The alert is sent only if the Pod was started in the Namespace named `vip` (Very Important Pod)

1. Select log messages that are associated with this event

    ```txt
    resource.type=gke_cluster
    jsonPayload.message="Started container"
    jsonPayload.metadata.namespace="vip"
    ```

    > Note: There will be no messages, as there is no such namespace yet!

1. Let's create the VIP Namespace

    ```shell
    kubectl create ns vip
    ```

1. Now start `nginx` pod and verify that message arrived to Stackdriver

    ```shell
    kubectl run nginx --image=nginx --namespace=vip
    ```

---

Next: [Namespaces RBAC and IAM](12-namespaces-rbac.md)
