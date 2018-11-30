# Pods and Deployments

## Module objectives

- Convert pod to a deployment
- Scale/update/rollout/rollback the deployment
- Define custom health checks and livenes probes
- Use horizontal pod autoscaler
- Use jobs and cronjobs to schedule task execution

---

## Convert pod to a deployment

There are a couple of problems with our current setup.

1. If the application inside a pod crashes nobody will restart the pod.
1. We already use services to connect our pods to each other, but we don't have an efficient way to scale the pods behind a service and load balance traffic between pod instances.
1. We don't have an efficient way to update our deployment without a downtime. We also can't easily rollback to the previous version if we need to do so.

Deployments can help to eliminate all these issues. Let's convert all our 3 pods to deployments.

1. Delete all running pods.

1. Open `manifests/db.yml` file

1. Delete the following 2 lines
    ```
    apiVersion: v1 
    kind: Pod
    ``` 
1. Add 4 more spaces in the beggining of each remaning line. If you use vim for editing you can check [this](http://vim.wikia.com/wiki/Shifting_blocks_visually) link to learn how to easily shift blocks of text in vim.

1. Add the following block to the top of the file

    ```
    apiVersion: extensions/v1beta1 
    kind: Deployment
    metadata:
      name: db
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gceme
          role: db
      template:
    ```
    Now your pod definition become a template for the newely created deployment. 

1. Repeat this steps for the `frontend` and `backend` pods. Don't forget to change the name of the deployment and  `machLabels` element.

1. Apply all 3 deployments

1. List deployments 

    ```
    $ kubectl get deployments
    NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    backend    1         1         1            0           1m
    db         1         1         1            1           1m
    frontend   1         1         1            1           6s
    ```

1. List replica sets

    ```
    $ kubectl get rs
    NAME                  DESIRED   CURRENT   READY     AGE
    backend-5c46cc4bb9    1         1         1         12m
    db-77df47c5dd         1         1         1         12m
    frontend-654b5ff445   1         1         1         11m
    ```

1. List pods

    ```
    $ kubectl get pod
    NAME                        READY     STATUS    RESTARTS   AGE
    backend-5c46cc4bb9-wf7qj    1/1       Running   0          12m
    db-77df47c5dd-ssgs6         1/1       Running   0          12m
    frontend-654b5ff445-b6fgs   1/1       Running   0          11m
    ```

1. List services and connect to the frontend LoadBalancer service. Verify that the app is working fine.

## Scale/update/rollout/rollback the deployment

1. Edit `manifests/backend.yml` update number of replicas to 3 and apply changes.

1. In your browser refresh the application several times. Make sure that 'Container IP' field sometimes changes. This indicates that the request comes to a different backend instance.

1. Edit `sample-app/main.go` file. At line 59 change `version` variable to `1.0.1` and save the file.

1. Rebuild container image with a new tag.
    ```
    $ export IMAGE=gcr.io/$PROJECT_ID/sample-k8s-app:1.0.1 
    $ docker build . -t $IMAGE
    $ docker push $IMAGE
    ```
1. Update `manifests/backend.yml` to use the new version of the container image and apply the changes.

1. Run the following command to watch how the pods are rolled out in a real time.

    ```
    $ watch kubectl get pod 
    ```

1. Open the application in the browser and make sure the backend version is updated.

1. Run the following command to view the deployment rollout history 
    ```
    $ kubectl rollout history deployment/backend
    deployments "backend"
    REVISION  CHANGE-CAUSE
    1         <none>
    2         <none>
    3         <none>
    ```

1. Rollback to the previous revisions

    ```
    $ kubectl rollout undo deployment/backend
    ```

1. Make sure the application now shows version `1.0.0` instead of `1.0.1`

## Define custom health checks and livenes probes

By default, kubernetes assumes that a pod is ready to accept requests as soon as its container is ready and the main process starts running. At this point of time, kubernetes services can redirect traffic to the pod. This can cause problems if the application needs some time to initialize. Let's reproduce this behavior.

1. Add `-delay=60` parameter to the backend startup command and apply changes. This will cause our app to sleep for a minute on startup.

1. Open the app in the web browser, for about a minute you should see the following error

    ```
    Error: Get http://backend:8080: dial tcp 10.51.249.99:8080: connect: connection refused
    ```
   
Now let's fix this problem by introducing a custom `readinesProbe` and `livenessProbe`. The first one tells kubernetes what command or HTTP request it can execute to check that the application is ready. The second one indicates a command or request that kubernetes should periodically run to check that a container is still healthy.


1. Edit `manifests/backend.yml`,  add the following section into it and apply the changes. 

    ```
            readinessProbe:
              httpGet:
                path: /healthz
                port: 8080
            readinessProbe:
              httpGet:
                path: /healthz
                port: 8080
    ```
    This 2 sections should go under `spec -> template -> spec -> containers[name=backend]` and should be aligned  together with `image`, `env` and `command` properties.

1. Run `watch kubectl get pod` to see how kubernetes is rolling out your pods. This time it will do it much slower, making sure that previous pods are ready before starting updating a new set of pods. 

## Use horizontal autoscaler

Now we will try to use horizontal autoscaler to automatically set number of backend instances based on the current load.

1. Scale the number of backend instances to 1 (You can either modify `manifests/backend.yml` file and apply changes or use `kubectl scale` command)

1. Apply autoscaling to the backend deployment. 

    ```
    kubectl autoscale deployment backend --cpu-percent=50 --min=1 --max=3
    ```

1. Check the status of the autoscaler.

    ```
    $ kubectl get hpa
    NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    backend   Deployment/backend   0%/50%    1         3         1          1m
    ```
1. Exec inside the pod

    ```
    $ kubectl exec -it <backend-pod-name> bash
    ```

1. Install `stress` and use the following command to generate some load.

    ```
    $ apt-get update
    $ apt-get install stress
    $ stress --cpu  20 --timeout 200
    ```

1. In a different terminal window watch autoscaler status

    ```
    $ watch kubectl get hpa
    NAME      REFERENCE            TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
    backend   Deployment/backend   149%/50%   1         3         3          13m
    ```
    Wait untill the autoscaler scales the number of backend pods to 3.

1. Save autoscaler definition as a kubernetes object and examine its content.

    ```
    $ kubectl get hpa -o yaml > autoscaler.yml
    ```

## Use jobs and cronjobs to schedule task execution

Sometimes there is a need to run one-off tasks. You can use pods to do that, but if the task fails nobody will be able to track that and restart the pod. Kubernetes jobs provide a better alternative to run one-off tasks. Jobs can be configured to retry failed task several times. If you need to run a job on a regular basis you can use Cron Jobs. Now let's create a Cron Job that will do a database backup for us.

1. Save the following file as `manifests/backup.yml` and apply the changes.

    ```
    apiVersion: batch/v1beta1
    kind: CronJob
    metadata:
      name: backup
    spec:
      schedule: "*/1 * * * *"
      jobTemplate:
        spec:
          backoffLimit: 2
          template:
            spec:
              containers:
              - name: backup
                image: mysql:5.6 
                command: ["/bin/sh", "-c"]
                args:
                - mysqldump -h db -u root -p$MYSQL_ROOT_PASSWORD sample_app
                env:
                - name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysql
                      key: password
              restartPolicy: OnFailure
    ```
    Here we are creating a cron job that runs each minute. `backoffLimit` is set to 2 so the job will be retried 2 times in case of failure. The job runs `mysqldump` command that prints the content of the `sample_app` database to stdout so it will be available in the pod logs. (And yes, I know that pod logs is the worst place to store a database backup :) )

1. Get cron job status
    ```
    $ kubectl get cronjob backup
    NAME      SCHEDULE      SUSPEND   ACTIVE    LAST SCHEDULE   AGE
    backup    */1 * * * *   False     0         2m              25m
    ```

1. After 1 minute a backup pod will be created, if you list the pods you should see the following

    ```
    $ kubectl get pod
    NAME                        READY     STATUS      RESTARTS   AGE
    backend-dc656c878-v5fh7     1/1       Running     0          3m
    backup-1543535520-ztrpf     0/1       Completed   0          1m
    db-77df47c5dd-d8zc2         1/1       Running     0          1h
    frontend-654b5ff445-bvf2j   1/1       Running     0          1h
    ```

1. View backup pod logs and make sure that backup was completed successfully.

    ```
    $ kubectl logs <backup-pod-name>
    ```

1. Delete the cron job

    ```
    $ kubectl delete cronjob backup
    ```

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
    #!/bin/bash

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
        mysql -u root  -e "update user set password=PASSWORD(\"$mynewpassword\") where User='root';"
        # stopping mysql because we have to restart after setting root password
        if ! kill -s TERM "$pid" || ! wait "$pid"; then
            echo >&2 'MySQL init process failed.'
            exit 1
        fi
    fi
    mysqld --wsrep-cluster-address=gcomm://$node_list &
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
              initialDelaySeconds: 15
              timeoutSeconds: 5
              successThreshold: 2
    ```
