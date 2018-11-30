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

## Convert persisten volume to persistent volume claim

Persistent volume claim automates disk provisioning.

1. Create Persistent Volume Claim for 50Gi disk

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: dynamic-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 50Gi
    ```

1. Apply configuration

    ```shell
    kubectl apply -f manifests/pvc.yaml
    ```

1. Verify PVC is created

    ```shell
    $ describe pvc/dynamic-data
    ..
    Events:
      Type       Reason                 Age   From                         Message
      ----       ------                 ----  ----                         -------
      Normal     ProvisioningSucceeded  90s   persistentvolume-controller  Successfully provisioned volume pvc-5c3004bd-f476-11e8-aef3-42010a840052 using kubernetes.io/gce-pd
    ```

1. Let's look at the volume

    ```shell
    $ kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
    pvc-5c3004bd-f476-11e8-aef3-42010a840052   50Gi       RWO            Delete           Bound    default/dynamic-data   standard                2m
    ```

1. You may find the disk in the gCloud output as well.

    ```shell
    $ gcloud compute disks list
    ..
    gke-gke-workshop-73cb6-pvc-5c3004bd-f476-11e8-aef3-42010a840052  europe-west1-d  50       pd-standard  READY
    ```

    Kubernetes provisioned 50Gi and made it available to use in Pods.

1. Reconfigure MySQL to use PVC instead of PV.

    The data will be lost - it is another disk!

    Edit `manifests/db.yaml` file

    ```yaml
    spec:
      volumes:
      - name: my-data
        persistentVolumeClaim:
          claimName: dynamic-data
      ```

1. You need to re-create the Pod or update the Deployment

    ```shell
    kubectl delete -f manifests/db.yaml
    kubectl apply -f manifests/db.yaml
    ```

1. Find the line in the event stream from the Pod telling that volume was mounted

    ```shell
    $ kubectl describe pod db
    ..
    Normal  SuccessfulAttachVolume  107s  attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-5c3004bd-f476-11e8-aef3-42010a840052"
    ```

1. Go to frontend and verify the previous notes are gone. That mean you use a new disk to store data.
