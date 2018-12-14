# Deployments and Health Checks

## Module Objectives

1. Convert pod to a deployment
1. Scale/update/rollout/rollback the deployment
1. Define custom health checks and liveness probes
1. Use horizontal pod autoscaler
1. Use jobs and cronjobs to schedule task execution

---

## Convert pod to a deployment

There are a couple of problems with our current setup.

* If the application inside a pod crashes, nobody will restart the pod.
* We already use services to connect our pods to each other, but we don't have an efficient way to scale the pods behind a service and load balance traffic between pod instances.
* We don't have an efficient way to update our deployment without a downtime. We also can't easily rollback to the previous version if we need to do so.

    **Deployments** can help to eliminate all these issues. Let's convert all our 3 pods to Deployments.

1. Delete all running pods.

    ```shell
    kubectl delete pod --all
    ```

1. Open the `manifests/db.yaml` file.

1. Delete the following 2 lines:

    ```yaml
    apiVersion: v1
    kind: Pod
    ```

1. Indent/add 4 more spaces in the beginning of each line. If you use vim for editing you can check [this](http://vim.wikia.com/wiki/Shifting_blocks_visually) link to learn how to easily shift blocks of text in vim.

1. Add the following block to the top of the file:

    ```yaml
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

    Now your pod definition becomes a template for the newly created deployment.

1. Repeat this steps for the `frontend` and `backend` pods. Don't forget to change the name of the deployment and the `machLabels` element.

1. Apply all 3 deployments.

1. List deployments.

    ```shell
    kubectl get deployments
    ```

    ```
    NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    backend    1         1         1            0           1m
    db         1         1         1            1           1m
    frontend   1         1         1            1           6s
    ```

1. List replica sets.

    ```shell
    kubectl get rs
    ```

    ```
    NAME                  DESIRED   CURRENT   READY     AGE
    backend-5c46cc4bb9    1         1         1         12m
    db-77df47c5dd         1         1         1         12m
    frontend-654b5ff445   1         1         1         11m
    ```

1. List pods.

    ```shell
    kubectl get pod
    ```

    ```
    NAME                        READY     STATUS    RESTARTS   AGE
    backend-5c46cc4bb9-wf7qj    1/1       Running   0          12m
    db-77df47c5dd-ssgs6         1/1       Running   0          12m
    frontend-654b5ff445-b6fgs   1/1       Running   0          11m
    ```

1. List services and connect to the frontend LoadBalancer service. Verify that the app is working fine.

    ```shell
    kubectl get services
    ```

## Scale/update/rollout/rollback the deployment

1. Edit `manifests/backend.yaml` update number of replicas to 3 and apply changes.

1. In your browser refresh the application several times. Make sure that the ‘Container IP' field sometimes changes. This indicates that the request comes to a different backend instance.

1. Edit `main.go` file. Change `version` to `1.0.1` and save the file.

1. Rebuild container image with a new tag.

    ```shell
    export IMAGE=gcr.io/$PROJECT_ID/sample-k8s-app:1.0.1
    docker build . -t $IMAGE
    docker push $IMAGE
    ```

1. Update `manifests/backend.yaml` to use the new version of the container image and apply the changes.

1. Watch how the pods are rolled out in real time.

    ```shell
    watch kubectl get pod
    ```

1. Open the application in the browser and make sure the backend version is updated.

1. View the deployment rollout history.

    ```shell
    kubectl rollout history deployment/backend
    ```

    ```
    deployments "backend"
    REVISION  CHANGE-CAUSE
    1         <none>
    2         <none>
    ```

1. Rollback to the previous revisions.

    ```shell
    kubectl rollout undo deployment/backend
    ```

1. Make sure the application now shows version `1.0.0` instead of `1.0.1`.

## Define custom health checks and liveness probes

By default, Kubernetes assumes that a pod is ready to accept requests as soon as its container is ready and the main process starts running. At this point of time, Kubernetes services can redirect traffic to the pod. This can cause problems if the application needs some time to initialise. Let's reproduce this behaviour.

1. Add `-delay=60` parameter to the backend startup command and apply changes. This will cause our app to sleep for a minute on startup.

1. Open the app in the web browser, for about a minute you should see an error.

    ```
    Error: Get http://backend:8080: dial tcp 10.51.249.99:8080: connect: connection refused
    ```

    Now let's fix this problem by introducing a custom `readinessProbe` and `livenessProbe`. The first one tells Kubernetes what command or HTTP request it can execute to check that the application is ready. The second one indicates a command or request that Kubernetes should periodically run to check that a container is still healthy.

1. Edit `manifests/backend.yaml`, add a probe section and apply the changes.

    ```yaml
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

1. Run `watch kubectl get pod` to see how Kubernetes is rolling out your pods. This time it will do it much slower, making sure that previous pods are ready before starting updating a new set of pods.

## Use horizontal autoscaling

Now we will try to use horizontal autoscaling to automatically set number of backend instances based on the current load.

1. Scale the number of backend instances to 1.

    > Note: You can either modify `manifests/backend.yaml` file and apply changes or use `kubectl scale` command

1. Apply autoscaling to the backend deployment.

    ```shell
    kubectl autoscale deployment backend --cpu-percent=50 --min=1 --max=3
    ```

1. Check the status of the autoscaler.

    ```shell
    kubectl get hpa
    ```

    ```
    NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    backend   Deployment/backend   0%/50%    1         3         1          1m
    ```

1. Exec inside the pod.

    ```shell
    kubectl exec -it <backend-pod-name> bash
    ```

1. Install `stress` and use the following command to generate some load.

    ```shell
    apt-get update & apt-get install stress
    stress --cpu 60 --timeout 200
    ```

1. In a different terminal window watch autoscaler status.

    ```shell
    watch kubectl get hpa
    ```

    ```
    NAME      REFERENCE            TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
    backend   Deployment/backend   149%/50%   1         3         3          13m
    ```

    Wait until the autoscaler scales the number of backend pods to 3.

1. Save autoscaler definition as a Kubernetes object and examine its content.

    ```shell
    kubectl get hpa -o yaml > manifests/autoscaler.yaml
    ```

## Use jobs and cronjobs to schedule task execution

Sometimes there is a need to run one-off tasks. You can use pods to do that, but if the task fails nobody will be able to track that and restart the pod. Kubernetes jobs provide a better alternative to run one-off tasks. Jobs can be configured to retry failed task several times. If you need to run a job on a regular basis you can use Cron Jobs. Now let's create a Cron Job that will do a database backup for us.

1. Save the following file as `manifests/backup.yaml` and apply the changes.

    ```yaml
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

    Here we are creating a cron job that runs each minute. `backoffLimit` is set to 2 so the job will be retried 2 times in case of failure. The job runs `mysqldump` command that prints the content of the `sample_app` database to stdout so it will be available in the pod logs. (And yes, pod logs is the worst place to store a database backup 😄 )

1. Get the cronjob status.

    ```yaml
    kubectl get cronjob backup
    ```

    ```
    NAME      SCHEDULE      SUSPEND   ACTIVE    LAST SCHEDULE   AGE
    backup    */1 * * * *   False     0         2m              25m
    ```

1. After 1 minute a backup pod will be created, if you list the pods you should see the backup completed.

    ```shell
    watch kubectl get pod
    ```

    ```
    NAME                        READY     STATUS      RESTARTS   AGE
    backend-dc656c878-v5fh7     1/1       Running     0          3m
    backup-1543535520-ztrpf     0/1       Completed   0          1m
    db-77df47c5dd-d8zc2         1/1       Running     0          1h
    frontend-654b5ff445-bvf2j   1/1       Running     0          1h
    ```

1. View backup pod logs and make sure that backup was completed successfully.

    ```shell
    kubectl logs <backup-pod-name>
    ```

1. Delete the cronjob.

    ```shell
    kubectl delete cronjob backup
    ```

---

Next: [Persistence](06-persistence.md)
