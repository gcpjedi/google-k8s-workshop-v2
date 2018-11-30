# Pods and Deployments

## Module objectives

1. Deploy the sample app to Kubernetes using pods
1. Use services and service discovery
1. Use LoadBalancer services
1. Use Secrets and ConfigMaps to externalize application credentials and configuration
1. Use sidecars and init containers
1. Use pod/node affinity and anti-affinity
1. Set pod limits

---

## Deploy the sample app to Kubernetes using pods

In this section, you will deploy the mysql database, `gceme` frontend and backend to Kubernetes using Kubernetes manifest files that describe the environment that the `gceme` binary/Docker image will be deployed to. They use the `gceme` Docker image that you've built in one of the previous modules.

1. First change directories to the sample-app:

    ```
    $ cd sample-app
    ```

1. Create the manifest to deploy MySQL database as manifests/db.yml:

    ```yaml
    apiVersion: v1 
    kind: Pod
    metadata:
      name: db
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: very-secret-password
        ports:
        - containerPort: 3306
          name: mysql
    ```

1. Deploy MySQL to Kubernetes

    ```
    $ kubectl apply -f manifests/db.yml
    ```

1. List all pods

    ```
    $ kubectl get pod
    NAME      READY     STATUS    RESTARTS   AGE
    db        1/1       Running   0          17s
    ```

1. Find out the mysql pod IP address.

    ```
    $ kubectl describe pod db | grep IP
    ```
    It is also useful to take a look at the full output of the `kubectl describe pod` command.

1. Create the manifest for the backend application, save it as `manifests/backend.yml` and deploy it to kubernetes using `kubectl apply` command.

    ```yaml
    kind: Pod
    apiVersion: v1 
    metadata:
      name: backend
    spec:
      containers:
      - name: backend
        image: <REPLACE_WITH_YOUR_OWN_IMAGE>
        command: ["app", "-mode=backend", "-run-migrations", "-port=8080", "-db-host=<REPLACE_WITH_MYSQL_IP>", "-db-password=very-secret-password" ]
        ports:
        - name: backend
          containerPort: 8080
    ```
    Don't forget to replace the image and the mysql ip address.

1. Find out the backend pod IP address in a similar way how we did it for mysql pod.

1. Create the manifest for the frontend application, save it as `manifests/frontend.yml` and deploy it to kubernetes using `kubectl apply` command.

    ```yaml
    kind: Pod
    apiVersion: v1
    metadata:
      name: frontend
    spec:
      containers:
      - name: frontend
        image: <REPLACE_WITH_YOUR_OWN_IMAGE>
        command: ["app", "-mode=frontend", "-backend-service=http://<REPLACE_WITH_BACKEND_IP>:8080", "-port=80"]
        ports:
        - name: frontend
          containerPort: 80
    ```

1. In the cloud console go to the 'Compute Engine' -> 'VM instances' page and ssh to any of the nodes. This is necessary because by default pods have only cluster-internal IP addresses and are not available from the outside.

1. From the node try to connect to the backend and frontend using curl
    ```
    curl <backend-ip>:8080
    curl <frontend-ip>

## Use services and service discovery

In the previous exercise, we manually copied IP addresses to connect pods. This is not only inconvenient - this will prevent your pods from reconnecting in case of failers because after a pod is restarted its IP address changes. Also in our current setup it is impossible to load balance the traffic between multiple instances of the backend pod. Services will help us to fix all of the above-mentioned issues.

1. In the `manifests/db.yml` file update `metadata section.  The whole section should look like the following.

    ```
    metadata:
      name: db
      labels:
        app: gceme
        role: db
    ```
    Use `kubectl apply` to apply the changes. Here we are adding labels to the backend pod. Services use labels under the hood to discover pod to which they should redirect traffic.

1. Create `manifests/db-svc.yml` with the following content and apply the changes.

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: db
    spec:
      type: ClusterIP
      ports:
        - port: 3306
      selector:
        app: gceme
        role: db
    ```


1. Update backend metadata.

    ```
    metadata:
      name: backend
      labels:
        app: gceme
        role: backend
    ```

1. Update backend startup command.
    ```
    command: ["app", "-mode=backend", "-run-migrations", "-port=8080", "-db-host=db", "-db-password=very-secret-password" ]
    ```

1. Redeploy the backend. Note that `kubectl apply` will not work for us this time, because we are updating pod startup command. Use `kuectl delete backend` to delete the pod and then recreate it.

1. Create and deploy `manifests/backend-svc.yml`

    ```
    kind: Service
    apiVersion: v1
    metadata:
      name: backend
    spec:
      ports:
      - name: http
        port: 8080
        targetPort: 8080
        protocol: TCP
      selector:
        role: backend
        app: gceme
    ```


1. Update frontend startup command
    ```
    command: ["app", "-mode=frontend", "-backend-service=http://backend:8080", "-port=80"]
    ```

1. Delete and recreate the frontend pod.

1. SSH to a worker node and check that the app is working.


## Use LoadBalancer services

Next thing we have to do is to expose the app to the external world. We can use the service of type LoadBalancer in order to do that.

1. Save the following file as `manifests/frontend-sv.yml` and use `kubectl apply` commad to create the frontend service.

    ```
    kind: Service
    apiVersion: v1
    metadata:
      name: frontend
    spec:
      type: LoadBalancer
      ports:
      - name: http
        port: 80
        targetPort: 80
        protocol: TCP
      selector:
        app: gceme
        role: frontend

    ```

1. Update frontend metadata and apply changes
    ```
    metadata:
      name: frontend
      labels:
        app: gceme
        role: frontend
    ```

1.  Run `kubectl get services` to list all services.

1. Retrieve the External IP for the frontend service: **This field may take a few minutes to appear as the load balancer is being provisioned**:

  ```
  $ kubectl get service frontend
  NAME             TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
  frontend         LoadBalancer   10.35.254.91   35.196.48.78   80:31088/TCP   1m
  ```

1. Copy the external ip and open it in your browser. Make sure that the application is working correctly.

1. In GCP Cloud Console, find and investigate the external IP address that the `LoadBalancer` service type created
    * VPC Network -> External IP addresses



## Use Secrets and ConfigMaps to externalize application credentials and configuration

One major problem with our current deployment is that we hardcode mysql root password in the pod configuration file. Usually, we want to externalize secrets and configuration from the kubernetes object definition. We can use secrets and config maps to do that.

 
1. Create a secret with the MySQL administrator password

    ```
    $ kubectl create secret generic mysql --from-literal=password=root
    secret/mysql created
    ```

1. Modify `env` section in the `manifests/db.yml` to look like the following.

    ```
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql
              key: password
    ```
    Here we are telling kubernetes to get the value for the MYSQL_ROOT_PASSWORD from the secret with name mysql. Each secret can have multiple key-value pairs, in our case we get the value from the password key.

1. Add exactly the same `env` section to the `manifests/backend.yml`

1. Modify the startup command in the `manifests/backend.yml` file 

    ```
    command: ["sh", "-c", "app -mode=backend -run-migrations -port=8080 -db-host=db -db-password=$MYSQL_ROOT_PASSWORD" ]
    ```
    As you can see, here we call `sh` instead of calling our app directly. Shell is required to do environment variable substitution for us. Also we use $MYSQL_ROOT_PASSWORD instead of hardcoded password.

1. Redeploy the backend and database. The database pod should be redeployed first, because the backend creates an empty database on startup and this database will be destroyed if you redeploy the database after the backend. Note, that you will not be able to use `kubectl apply` command this time. Instead, you should use `kubectl delete` first and then redeploy pod.  

1. Make sure that the app is still working fine.

## Use sidecars and init containers

On startup our backend pod creates a database for itself if it doesn't exist and run migrations. However, usually we want to externalize such tasks from the application pod. We can use init containers to do that. 

First, let's verify that the app will fail if we restart the db and didn't run migrations.

1. Delete `-run-migrations` parameters from the backend startup command.

1. Delete and recreate the backend pod. At this point the app should work fine because we are still using an old database.

1. Delete and recreate the db pod 

1. Open the app UI you should see the following error `Error 1049: Unknown database 'sample_app'`

Now let's fis the error by allowing the backend pod to run migrations each time before it is started.

1. Add the following section to the 'manifests/backend.yml'
    ```
      initContainers:
      - name: init-myservice
        image: <REPLACE_WITH_YOUR_OWN_IMAGE>
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql
              key: password
        command: ["sh", "-c", "app -run-migrations -port=8080 -db-host=db -db-password=$MYSQL_ROOT_PASSWORD" ]
    ```
    You can append this lines directly to the end of the file. `initContainers` section should be aligned with the same number of spaces as `containers` section    

1. Recreate the backend pod.

1. Make sure the app is working fine. 

## Use pod/node affinity and anti-affinity

By default the kubernetes scheduler is doing a great job by trying to evnly distribute pods between nodes, taking into consideration current node resource utilization as well as other criteria. But there are situations when you want to influence how pods are placed. For example, let's imagine that we want all our 3 pods to be colocated on a single node (this could make sense if we care more about performance than high availability) But we don't want to pick a particular node - instead we prefer the scheduler to do this for us. We also don't want this rule to be very strict - if there is not such a node that rave enough resources to host all of our pods we want to allow the scheduler to break this rule and put pods to different nodes. This is a kind of rule that can be expressed using [node/pod affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)

1. Run the following command to make sure that your pods are allocated on different worker nodes (most likely this will be the case)

    ```
    $ kubectl get pod -o wide
    NAME       READY     STATUS    RESTARTS   AGE       IP           NODE
    backend    1/1       Running   0          10m       10.48.1.28   gke-gke-workshop-0-default-pool-c70d26ac-gvpw
    db         1/1       Running   0          15m       10.48.1.26   gke-gke-workshop-0-default-pool-c70d26ac-gvpw
    frontend   1/1       Running   0          20h       10.48.0.11   gke-gke-workshop-0-default-pool-c70d26ac-p2x2
    ```
1. Add the following block to the `spec` element for all 3 of our pods. (this element should be aligned with the same number of spaces as `containers` key)

    ```
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - gceme
              topologyKey: "kubernetes.io/hostname"
    ``` 
    Refer to the [official documentation](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#inter-pod-affinity-and-anti-affinity-beta-feature) to make sure you understand the meaning of all specified properties.
1. Delete all pods 

    ```
    $ kubectl delete pod --all
    ```

1. Recreate all pods 
    ```
    $ kubectl apply -R -f manifests/
    ```

1. Make sure all pods are colocated on a single node now. 
    ```
    $ kubectl get pod -o wide
    NAME       READY     STATUS    RESTARTS   AGE       IP           NODE
    backend    1/1       Running   0          5m        10.48.1.30   gke-gke-workshop-0-default-pool-c70d26ac-gvpw
    db         1/1       Running   0          5m        10.48.1.29   gke-gke-workshop-0-default-pool-c70d26ac-gvpw
    frontend   1/1       Running   0          5m        10.48.1.31   gke-gke-workshop-0-default-pool-c70d26ac-gvpw
    ```

## Set pod limits

Currently, all our pods can consume as many resources as they want. This is rarely a good idea, because in this case your pods can influence somebody else's workloads. First, let's verify that right now this is the case.

1. Get inside the backend pod

    ```
    $ kubectl exec -i -t backend bash
    ``` 

1. Check how much memory is available

    ```
    $ cat /proc/meminfo             
    MemTotal:        3794356 kB
    MemFree:          864724 kB
    MemAvailable:    2665768 kB
    ...
    ```

1. Install `stress` utility inside the container

    ```
    $ apt-get update
    $ apt-get install stress
    ```

1. Try to consume 1GB of memory

    ```
    $ stress --vm-bytes 1g --vm-keep -m 1
    ```

1. In a different terminal window exec into the backend pod and run `top` command

    ```
    Tasks:   7 total,   2 running,   5 sleeping,   0 stopped,   0 zombie
    %Cpu(s): 99.0 us,  1.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
    KiB Mem :  3794356 total,   114472 free,  1937036 used,  1742848 buff/cache
    KiB Swap:        0 total,        0 free,        0 used.  1612368 avail Mem

      PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                                                                           
      237 root      20   0 1055860 1.000g    212 R 96.0 27.6   1:54.22 stress                                                                                                                            
        1 root      20   0    4292    732    656 S  0.0  0.0   0:00.00 sh                                                                                                                                
        5 root      20   0   52020   8520   6500 S  0.0  0.2   0:00.04 app
       10 root      20   0   18208   3200   2644 S  0.0  0.1   0:00.02 bash
      236 root      20   0    7280    896    816 S  0.0  0.0   0:00.00 stress
      238 root      20   0   18204   3392   2832 S  0.0  0.1   0:00.00 bash  
      244 root      20   0   41032   3124   2660 R  0.0  0.1   0:00.01 top
    ```
    
1. Stop `stress` and `top` commands and exit from the backend pod in both terminal windows

Now let's try to prevent the pod from consuming as much memory as it wants.

1. Add the following section to the `manifests/backend.yml`

    ```
        resources:
          limits:
            memory: "800Mi"
          requests:
            memory: "600Mi"
    ```
    We are defining resource limits for all containers individually, so this element should go under `spec -> containers[name=backend]` and should be aligned together with `image` and `command` properties

1. Delete and recreate the backend pod.

1. Try to run stress command inside the backend pod once again.

    ```
    $ stress --vm-bytes 1g --vm-keep -m 1
    stress: info: [226] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
    stress: FAIL: [226] (415) <-- worker 227 got signal 9
    stress: WARN: [226] (417) now reaping child worker processes
    stress: FAIL: [226] (451) failed run completed in 1s
    ```

## Optional Exercises

### Use sidecar containers 

A pod can host multiple containers, not just one. Let's try to extend the backend pod and add one more container into it. This pod can run any image, startup command should be `sleep 100000`. After the pod is ready try to exec into the second container and access backend app using `localhost`


