Monitoring and Logging
====================

Objectives
----------

- enable Stackdriver logging
- observe cluster logs
- setup alert

Enable logging
--------------

By default, when you create Kubernetes GKE cluster, the `--enable-cloud-logging` flag is automatically set.

Go to the GKE console to check this: https://console.cloud.google.com/kubernetes/clusters/details/europe-west1-d/gke-workshop

Make sure the "Stackdriver logging" checkbox is turned on.

View logs
---------

There are two types of logs: _container_ and _system_. Container logs are collected from the running containers. System logs are produced by cluster components like `kubelet` and `api`. There are also events like cluster creation that are produced by Google cloud.

View container logs

1. Go to the Logging dashboard of the Web Console.

1. Switch to advanced mode

    ![Advanced filter](img/logging-advanced-filter.png)

1. Edit the filter

    ```shell
    resource.type="container"
    resource.labels.namespace_id="default"
    resource.labels.cluster_name="gke-workshop"
    resource.labels.container_name="backend"
    ```

    You will see logs from the backend container.

    While editing the filter, you can use auto competition ("Control+Space") to explore possible options.

1. Find the record that shows backend container was started and ready to serve requests

    The line container prints is "Operating in backend mode..."

    The message goes to the `textPayload` field.

    Operator `include` is `:`

    Prepare the filter statement yourself.

1. Write a filter for audit operations on cluster

    ```json
    resource.type=gke_cluster
    resource.labels.cluster_name="gke-workshop"
    protoPayload.@type:AuditLog
    ```

    It will show you when cluster was created.

    Delete the second line. Now you will see operations for all the clusters in the account.

1. Some other interesting `resource.type` to explore are `k8s_cluster`, `k8s_node` and `k8s_pod`.

Setup alert
-----------

In this exercise you will setup alert on new pod started. The alert is sent only if pod was started in the namespace named `vip` (Very Importnat Pod).

1. Select log messages that are associated with this event

    ```txt
    resource.type=k8s_pod
    jsonPayload.message="Started container"
    jsonPayload.metadata.namespace="vip"
    ```

    No messages - as there is no such namespace yet!

1. Let's create such a namespace

    ```shell
    $ kubectl create ns vip
    namespace/vip created
    ```

1. Now start `nginx` pod and verify that message arrived to Stackdriver

    ```shell
    kubectl run nginx --image=nginx --namespace=vip
    ```

-- didn't manage to get log-based alert metrcis for me
