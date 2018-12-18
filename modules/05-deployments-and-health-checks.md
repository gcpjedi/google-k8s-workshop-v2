# Deployments and Health Checks

## Module Objectives

- Convert a pod to a deployment
- Scale/update/rollout/rollback the deployment
- Define custom health checks and liveness probes
- Use horizontal pod autoscaler
- Use jobs and cronJobs to schedule task execution

---

## Convert a Pod to a Deployment

There are a couple of problems with our current setup.

* If the application inside a Pod crashes, nobody will restart the Pod.
* We already use Services to connect our Pods to each other, but we don't have an efficient way to scale the Pods behind a Service and load balance traffic between Pod instances.
* We don't have an efficient way to update our Pods without a downtime. We also can't easily rollback to the previous version if we need to do so.

    **Deployments** can help to eliminate all these issues. Let's convert all our 3 Pods to Deployments.

1. Delete all running Pods.

    ```shell
    kubectl delete pod --all
    ```

1. Open the `manifests/db.yaml` file.

1. Delete the top 2 lines.

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

    Your Pod definition becomes a template for the newly created Deployment.

1. Repeat this steps for the `frontend` and `backend` Pods. Don't forget to change the name of the Deployment and the `machLabels` element.

1. List Deployments.

    ```shell
    kubectl get deployments
    ```

    ```
    NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    backend    1         1         1            0           1m
    db         1         1         1            1           1m
    frontend   1         1         1            1           6s
    ```

1. List ReplicaSets.

    ```shell
    kubectl get rs
    ```

    ```
    NAME                  DESIRED   CURRENT   READY     AGE
    backend-5c46cc4bb9    1         1         1         12m
    db-77df47c5dd         1         1         1         12m
    frontend-654b5ff445   1         1         1         11m
    ```

1. List Pods.

    ```shell
    kubectl get pod
    ```

    ```
    NAME                        READY     STATUS    RESTARTS   AGE
    backend-5c46cc4bb9-wf7qj    1/1       Running   0          12m
    db-77df47c5dd-ssgs6         1/1       Running   0          12m
    frontend-654b5ff445-b6fgs   1/1       Running   0          11m
    ```

1. List Services and connect to the `frontend` LoadBalancer service's IP in a web browser. Verify that the app is working fine.

    ```shell
    kubectl get services
    ```

## Scale/Update/Rollout/Rollback the Deployment

    ```shell
    kubectl get services
    ```

### Scale the Deployment

1. Edit `manifests/backend.yaml`. Update the number of replicas to 3 and apply changes.

1. In your browser refresh the application several times. Notice that `Container IP` field sometimes changes. This indicates that the request comes to a different backend Pod.

### Update and Rollout the Deployment

1. Edit `main.go` file. Change `version` to `1.0.1` and save the file. (Around line 60)

1. Rebuild the container image with a new tag.

    ```shell
    export IMAGE=gcr.io/$PROJECT_ID/sample-k8s-app:1.0.1
    docker build . -t $IMAGE
    docker push $IMAGE
    ```

1. Update `manifests/backend.yaml` to use the new version of the container image and apply the changes.

    Replace the image version in both the `initContainers` and `containers` sections.

    ```yaml
    image: gcr.io/<PROJECT-ID>/sample-k8s-app:1.0.1
    ```

    > Note: replace `<PROJECT-ID>` with your project id.

1. Watch how the Pods are rolled out in real time.

    ```shell
    watch kubectl get pod
    ```

1. Open the application in the browser and make sure the `backend` version is updated.

1. View the Deployment rollout history.

    ```shell
    kubectl rollout history deployment/backend
    ```

    ```
    deployments "backend"
    REVISION  CHANGE-CAUSE
    1         <none>
    2         <none>
    ```

### Rollback the Deployment

1. Rollback to the previous revision

    ```shell
    kubectl rollout undo deployment/backend
    ```

1. Make sure the application now shows version `1.0.0` instead of `1.0.1`.

## Define Custom Health Checks and Liveness Probes

By default, Kubernetes assumes that a Pod is ready to accept requests as soon as its container is ready and the main process starts running. Once this condition is met, Kubernetes Services will redirect traffic to the Pod. This can cause problems if the application needs some time to initialize. Let's reproduce this behavior.

1. Add `-delay=60` parameter to the `backend` startup command and apply changes. This will cause our app to sleep for a minute on startup.

1. Open the app in the web browser. You should see the following error for about a minute:

    ```
    Error: Get http://backend:8080: dial tcp 10.51.249.99:8080: connect: connection refused
    ```

Now let's fix this problem by introducing a `readinessProbe` and a `livenessProbe`.

The `readinessProbe` and `livenessProbe` are used by Kubernetes to check the health of the containers in a Pod. The probes can check either HTTP endpoints or run shell commands to determine the health of a container. The difference between readiness and liveness probes is subtle, but important.

If an app fails a `livenessProbe`, kubernetes will restart the container.

> Caution: A misconfigured `livenessProbe` could cause deadlock for an application to start. If an application takes more time than the probe allows, then the livenessProbe will always fail and the app will never successfully start. See [this document](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) for how to adjust the timing on probes.

If an app fails a `readinessProbe`, Kubernetes will consider the container unhealthy and send traffic to other Pods.

> Note: Unlike liveness, the Pod is not restarted.

For this exercise, we will use only a `readinessProbe`.

1. Edit `manifests/backend.yaml`, add the following section into it and apply the changes.

    ```yaml
            readinessProbe:
              httpGet:
                path: /healthz
                port: 8080
    ```

    This section should go under `spec -> template -> spec -> containers[name=backend]` and should be aligned with `image`, `env` and `command` properties.

1. Run `watch kubectl get pod` to see how Kubernetes is rolling out your Pods. This time it will do it much slower, making sure that previous Pods are ready before starting a new set of Pods.

## Use Horizontal Autoscaler

Now we will use the horizontal autoscaler to automatically set the number of backend instances based on the current load.

1. Scale the number of backend instances to 1.

    > Note: You can either modify `manifests/backend.yaml` file and apply changes or use `kubectl scale` command.

1. Apply autoscaling to the backend Deployment.

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

1. Exec inside the Pod.

    ```shell
    kubectl exec -it <backend-pod-name> bash
    ```

1. Install `stress` and use the following command to generate some load.

    ```shell
    apt-get update & apt-get install stress
    stress --cpu 60 --timeout 200
    ```

1. In a different terminal window watch the autoscaler status.

    ```shell
    watch kubectl get hpa
    ```

    ```
    NAME      REFERENCE            TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
    backend   Deployment/backend   149%/50%   1         3         3          13m
    ```

    Wait until the autoscaler scales the number of `backend` Pods to 3.

1. Save the autoscaler definition as a Kubernetes object and examine its content.

    ```shell
    kubectl get hpa -o yaml > manifests/autoscaler.yaml
    ```

## Use Jobs and CronJobs to Schedule Task Execution

Sometimes there is a need to run one-off tasks. You can use Pods to do that, but if the task fails nobody will be able to track that and restart the Pod. Kubernetes Jobs provide a better alternative to run one-off tasks. Jobs can be configured to retry failed task several times. If you need to run a Job on a regular basis you can use CronJobs. Now let's create a CronJob that will do a database backup for us.

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

    This creates a CronJob that runs each minute. `backoffLimit` is set to 2 so the job will be retried 2 times in case of failure. The job runs the `mysqldump` command that prints the contents of the `sample_app` database to stdout. This will be available in the Pod logs. (And yes, I know that Pod logs is the worst place to store a database backup 😄  )

1. Get the cronjob status.

    ```yaml
    kubectl get cronjob backup
    ```

    ```
    NAME      SCHEDULE      SUSPEND   ACTIVE    LAST SCHEDULE   AGE
    backup    */1 * * * *   False     0         2m              25m
    ```

1. After 1 minute a` backup` Pod will be created, if you list the Pods you should see the backup completed.

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

1. View the `backup` Pod logs and make sure that backup was completed successfully.

    ```shell
    kubectl logs <backup-pod-name>
    ```

1. Delete the CronJob.

    ```shell
    kubectl delete cronjob backup
    ```

---

Next: [Persistence](06-persistence.md)
