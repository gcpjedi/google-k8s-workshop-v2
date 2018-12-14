# Pesistence

## Module Objectives

1. Use persistent volumes to store persistent data, map the volume to the container
1. Convert persisten volume to persistent volume claim
1. Create a storage classes
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

## Create a storage class

Google cloud offers several disk types. They differ based on performance, reliability and price. How can I use particular disk type for Kubernetes workloads?

Kubernetes defines resource type called StorageClass. By specifying StorageClass in you PVCs you can provision different disk types.

1. Create a storage class that uses SSDs

    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: ssd
    provisioner: kubernetes.io/gce-pd
    parameters:
      type: pd-ssd
    ```

    `type: pd-ssd` tells `gce-pd` provisioner to create an SSD disk instead of standard one.

1. Apply coniguration

    ```shell
    kubectl apply -f manifests/storage-class.yaml
    ```

1. Verify the StorageClass was created

    ```shell
    $ kubectl get storageClass
    NAME                 PROVISIONER            AGE
    ssd                  kubernetes.io/gce-pd   5m
    standard (default)   kubernetes.io/gce-pd   1h
    ```

1. Now create PVC that uses this class

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: fast-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 50Gi
      storageClassName: ssd
    ```

    ```shell
    kubectl apply -f manifests/pcv-fast.yaml
    ```

    ```shell
    $ kubectl get pvc
    NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    dynamic-data   Bound    pvc-5c3004bd-f476-11e8-aef3-42010a840052   50Gi       RWO            standard       38m
    fast-data      Bound    pvc-eaac23d7-f47a-11e8-aef3-42010a840052   50Gi       RWO            ssd            5m
    ```

1. The last step is to change PVC reference from the database Pod and update the deployment. Please do this by yourself.

## Converting mysql deployment to a statefull set

When you are using deployments to manage your pods it is very easy to scale stateless applications, like our backend and frontend, but scaling statefull applications is more difficult. If you want to scale a mysql database you have to start the first node in a bootstrap mode, then you have to wait until this node is redy and finally you have to start all other nodes and add them to the cluster. If you try to use deployment to scale your database it will create 3 instances of the database pod simultaneously and you will not be able to create a cluster. Statefull Set is the right object to do such kund of deployment. Now let's use a Statefull Set to convert our mysql pod to a 3-node galera mysql cluster.  

1. In the `sample-app` folder create a subfolder `mysql-galera`

1. Inside `mysql-galera` folder create a Dockerfile with the following content 

    ```
    FROM ubuntu:16.04
    ENV DEBIAN_FRONTEND noninteractive

    RUN apt-get update
    RUN apt-get install -y software-properties-common
    RUN apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 BC19DDBA
    RUN add-apt-repository 'deb http://releases.galeracluster.com/mysql-wsrep-5.7/ubuntu xenial main'
    RUN add-apt-repository 'deb http://releases.galeracluster.com/galera-3/ubuntu xenial main'
    RUN apt-get update
    RUN apt-get install -y galera-3 galera-arbitrator-3 mysql-wsrep-5.7 rsync lsof host
    COPY my.cnf /etc/mysql/my.cnf
    COPY start.sh  /start.sh
    ENTRYPOINT ["/start.sh"]
    ```

    This dockerfile simply installs Galera using the Codership repository and copies the my.cnf start.sh over.  

1. Add `my.cnf` file to the `mysql-galera` folder

    ```
    [mysqld]
    user = mysql
    bind-address = 0.0.0.0
    wsrep_provider = /usr/lib/galera/libgalera_smm.so
    wsrep_sst_method = rsync
    default_storage_engine = innodb
    binlog_format = row
    innodb_autoinc_lock_mode = 2
    innodb_flush_log_at_trx_commit = 0
    query_cache_size = 0
    query_cache_type = 0
    ```
    This file configures galera replication.

1. Add `start.sh` file to the `mysql-galera` folder. 

    ```
    #!/bin/bash -e

    mkdir /var/run/mysqld
    chown mysql:mysql /var/run/mysqld

    node_list=$(host $SERVICE_NAME | grep "has address" |  awk '{print $4}' | paste -s -d, -)

    if [ -z "$node_list" ]; then
        # if we are in a bootstrap mode we want to set root password first
        mysqld --wsrep-cluster-address=gcomm://$node_list &
        pid="$!"
        # waiting for mysql to become ready
        while !(mysqladmin ping)
        do
           sleep 3
           echo "waiting for mysql ..."
        done
        # setting root password to $MYSQL_ROOT_PASSWORD
        mysql -u root  -e "use mysql; update user set authentication_string=password(\"$MYSQL_ROOT_PASSWORD\") where user='root';"
        # stopping mysql because we have to restart after setting root password
        if ! kill -s TERM "$pid" || ! wait "$pid"; then
            echo >&2 'MySQL init process failed.'
            exit 1
        fi
    fi
    mysqld --wsrep-cluster-address=gcomm://$node_list 
    ```
    `$SERVICE_NAME` is the DNS name of the service that wraps all 3 mysql nodes. This is a headles service, so instead of a virtual IP address it resolves to the list of all IP addresses of the underlying pods. When the first node is started `host $SERVICE_NAME` command returns and empty list. In this case `node_list` variable is empty. After the first node is started the command returns its IP address, after the second one is started it returns 2 IP addresses separated by comma. This is exactly what we need to start the galera cluster: the first node should be started with `--wsrep-cluster-address=gcomm://` parameter, the secon with `--wsrep-cluster-address=gcomm://<first-node-ip>`, the third with `--wsrep-cluster-address=gcomm://<first-node-ip>,<second-node-ip>` and so on.

1. Make `start.sh` executable
    ```
    chmod +x start.sh
    ```

1. Build the image and push it to the container registry
    ```
    $ docker build . -t gcr.io/$PROJECT_ID/mysql-galera
    $ docker push gcr.io/$PROJECT_ID/mysql-galera
    ```

1. Delete db deployment and db service
    ```
    $ kubectl delete deployment db
    $ kubectl delete svc db
    ```

1. Save the following file as `mysql-galera-svc.yml` and apply the changes

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: galera-cluster
    spec:
      ports:
      - port: 3306
        name: mysql
      clusterIP: None
      selector:
        app: gceme
        role: db
      publishNotReadyAddresses: true
    ```
    The most inportant parameter here is `clusterIP: None` This is called a [Headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) In this case kubernes will not create a cluster IP for the service and DNS name `galera-cluster` will point directly to the list of undrlying pods.

1. Save the following file as `mysql-galera.yml` and apply the changes

    ```
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: db
    spec:
      serviceName: "galera"
      replicas: 3
      selector:
        matchLabels:
          app: gceme
          role: db
      template:
        metadata:
          labels:
            app: gceme
            role: db
        spec:
          containers:
          - name: mysql
            image: <REPLACE_WITH_YOUR_OWN_MYSQL_IMAGE> 
            ports:
            - containerPort: 3306
              name: mysql
            - containerPort: 4444
              name: sst
            - containerPort: 4567
              name: replication
            - containerPort: 4568
              name: ist
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
            - name: SERVICE_NAME
              value: galera-cluster
            readinessProbe:
              exec:
                command:
                - sh
                - -c
                - "mysql -u root -p$MYSQL_ROOT_PASSWORD -e 'show databases;'"
              initialDelaySeconds: 5
              timeoutSeconds: 2
              successThreshold: 2
    ```
1. Connect to the app and check that you can add notes .

1. Exec inside one of the db pods

    ```
    $ kubectl exec -it db-0 bash
    ```
1. Connect to the mysql

    ```
    $ mysql -u root -p$MYSQL_ROOT_PASSWORD sample_app
    ```
1. Check that your notes are there
    ```
    mysql> select * from notes;
    +----+---------------------+---------------------+------------+---------------------+
    | id | created_at          | updated_at          | deleted_at | note                |
    +----+---------------------+---------------------+------------+---------------------+
    |  1 | 2018-11-30 09:09:32 | 2018-11-30 09:09:32 | NULL       | where are you from? |
    |  2 | 2018-11-30 09:09:48 | 2018-11-30 09:09:48 | NULL       | I am from Finland   |
    +----+---------------------+---------------------+------------+---------------------+
    2 rows in set (0.00 sec)
    ``` 

## Optional Exercises

### Use persistent columes in the stateful set 

Adopt `manifests/mysql-galera.yml` to use persistent volume claims. Make suer the data survives after you delete and recreate the database.
