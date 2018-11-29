Pods and Deployments
=============

Module objectives
-----------------

- Convert pod to a deployment
- Scale/update/rollout/rollback the deployment
- Define custom health checks and livenes probes
- Use horizontal and vertical pod autoscaler and verify it works correctly
- Use jobs and cronjobs to schedule task execution

---

Convert pod to a deployment
---------------------------

There are a couple of problems with our curent setup.

1. If the application inside a pod crashes nobody will restart the pod.
1. We already use services to connect our pods to each other, but we don't have an efficient way to scale the pods behind a service and load balance trafic between pod instances.
1. We don't have an efficient way to update our deployment without a downtime. We also can't easliy rollback to the previous ersion if we need to do so.

Deployments can help to eliminate all this issues. Let's convert all our 3 pods to deployments.

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

1. Repet this steps for the `frontend` and `backend` pods. Dotn' forget to change name of the deployment and  `machLabels` element.

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

Scale/update/rollout/rollback the deployment
--------------------------------------------

1. Edit `manifests/backend.yml` update number of replicas to 3 and apply changes.

1. In your browser refresh the application several times. Make sure that 'Container IP' field sometimes changes. This indicates that request comes to a different backend instance.

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

Define custom health checks and livenes probes
----------------------------------------------

By default kubernetes asumes that 
