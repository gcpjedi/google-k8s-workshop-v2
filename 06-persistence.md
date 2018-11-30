# Pesistence

## Module objectives

1. Use persistent volumes to store persistent data, map the volume to the container
1. Convert persisten volume to persistent volume claim
1. Create storage classes
1. Use statefull set to deploy a mysql galera cluster

## Use persistent volume to store data

1. Create a data disk

    ```shell
    gcloud compute disks create \
      --size=50GB \
      --zone=europe-west1-d \
      my-data-disk
    ```

    The disk must be in the same zone and project as GKE cluster.

1. Edit the database manifest to add volume to the Pod spec

    ```yaml
    spec:
      volumes:
      - name: my-data
        gcePersistentDisk:
          pdName: my-data-disk
          fsType: ext4
    ```

    Note to add it to the Pod spec not the Deployment spec!

1. Now mount the volume inside MySQL container

    ```yaml
    containers:
      - image: mysql:5.6
        name: mysql
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: my-data
    ```

1. Re-create the Deployment

  ```shell
  kubectl apply -f manifests/db.yaml
  ```

1. Test the data persists between Pod restarts

    Add some notes to the frontend app.

    Then delete the mysql pod.

    Deployment controller will automatically re-create the Pod and mount the persistent volume.
