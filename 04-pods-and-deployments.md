Pods and Deployments
=============

Module objectives
-----------------

1. Deploy our own image to the k8s cluster using pod
1. Use Secrets and ConfigMaps to externalize application credentials and configuration, map them to the pod using volumes or environment variables
1. Use sidecars and init containers
1. Assign pods to specific nodes/node pools
1. Set pod limits
1. Convert pod to a deployment
1. Scale/update/rollout/rollback the deployment
1. Verify that individual pods in a deployment are restarted after failures
1. Define custom health checks and livenes probes
1. Use horizontal and vertical pod autoscaler and verify it works correctly
1. Use jobs and cronjobs to schedule task execution

---

Deploy the sample app to Kubernetes using pods
---------------------------------------------

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

Use services and service discovery
---------------------------------

In the previous exercise we manually copied IP addresses to connect pods. This is not only inconvinient - this will prevent your pods fron reconnecting in case of failers, because after a pod is restarted its IP address cahnges. Also in our curent setup it is imposible to load balance the trafic between multiple instances of the backend pod. Services will help us to fix all of the above mentioned issues.

1. In the `manifests/db.yml` file update `metadata section.  The whole seciton should look like the following.

    ```
    metadata:
      name: db
      labels:
        app: gceme
        role: db
    ```
    Use `kubectl apply` to apply the changes. Here we are adding labels to the backend pod. Services use labels under the hood to discover pod to which they should redirect trafic.

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


Using LoadBalancer services
---------------------------

Next thing we have to do is to expose the app to the external world. We can use service of type LoadBalancer in order to do that.

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



Use Secrets and ConfigMaps to externalize application credentials and configuration
-----------------------------------------------------------------------------------

One major problem with our curent deployment is that we hardcode mysql root passowrd in the pod configuration file. Usually we want to externalize secrets and cofniguration from the kubernetes object definition. We can use secrets and config maps to do that.

 
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
    Here we are telling kubernetes to get the value for the MYSQL_ROOT_PASSWORD from the secret with name mysql. Each secret can have multiple key value pairs, in our case we get the value from the password key.

1. Add exactly the same `env` section to the `manifests/backend-pod.yml`

1. Modify the startup command in the `manifests/backend-pod.yml` file 

    ```
    command: ["sh", "-c", "app -mode=backend -run-migrations -port=8080 -db-host=<REPLACE_WITH_MYSQL_IP> -db-password=$MYSQL_ROOT_PASSWORD" ]
    ```
    As you can see, here we call `sh` instead of calling our app directly. Shell is required to do environment variable substitution for us. Also we use $MYSQL_ROOT_PASSWORD instead of hardcoded password.

1. Redeploy the backend and database. The database pod should be redeployed first, because the backend creates an empty database on startup and this database will be destroyed if you redeploy the database after the backend. Note, that you will not be able to use `kubectl apply` command this time. Instead you should use `kubectl delete` first and then redeploy pod. Pod ip addresses may change so you will have to update all 3 deployment files. 

1. Make sure that the app is still working fine.

Use sidecars and init containers
--------------------------------

On startup our sample application creates a database for itself if it doesn't exist and run migrations. However, usually we want to externalize such tasks from the application pod. We can use init containers to do that. 

1. Add the following section to the 'manifests/db.yml'
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
      command: ["sh", "-c", "app -run-migrations -port=8080 -db-host=<REPLACE_WITH_MYSQL_IP> -db-password=$MYSQL_ROOT_PASSWORD" ]
    ```
    You can append this lines directly to the end of the file. `initContainers` section should be aligned with the same number of spaces as `containers` section    

1. Remove `-run-migrations` command from  the backend startup command
