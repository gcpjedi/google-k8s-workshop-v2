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

    ![](img/logging-advanced-filter.png)

1. Edit the filter

    ```shell
    resource.type="container"
    resource.labels.namespace_id="default"
    resource.labels.cluster_name="gke-workshop"
    resource.labels.container_name="backend"
    ```

    You will see logs from the backend container.

    While editing the filter, you can use auto competition ("Control+Space") to explore possible options.

1. Find the record that shows frontend app was started and ready to serve requests

    View system logs

1. Switch back to basic mode

1. Select "GKE Cluster Operations -> gke-workshop"

1. Find the record that tells you which node db master pod was scheduled

You can convert the filter to advanced mode and look at it:

```shell
resource.type="gke_cluster"
resource.labels.cluster_name="gke-workshop"
```

View platform events
--------------------

You can view the events Google cloud associate with your cluster.

1. Select "GKE Cluster Operations -> gke-workshop"

1. Select log type: "activity"

1. Find the record which tells you when the cluster was started.

## Setup alert

In this exercise you will setup alert on new pod started. The alert is sent only if pod has label `status=vip` (Very Importnat Pod).
