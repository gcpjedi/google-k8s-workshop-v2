# Services

## Module Objectives

1. Recreate the sample application using different Service types
1. Modify a Service and track iptables changes

---

## Recreate the Sample Application Using Different Service Types

1. Delete everything from the default Namespace

    ```shell
    kubectl delete deployment --all
    kubectl delete svc --all
    kubectl delete statefulset --all
    kubectl delete hpa --all
    ```

1. Redeploy the sample app (minimal version). Save the following as `manifests/sample-app.yaml` and apply the changes

    > Note: Replace the image with your own, e.g. `gcr.io/$PROJECT_ID/sample-k8s-app:1.0.0`

    ```yaml
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
    ---
    apiVersion: apps/v1
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
        metadata:
          name: db
          labels:
            app: gceme
            role: db
        spec:
          containers:
          - image: mysql:5.6
            name: mysql
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
            ports:
            - containerPort: 3306
              name: mysql
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: backend
    spec:
      ports:
      - name: http
        port: 8080
      selector:
        role: backend
        app: gceme
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: backend
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gceme
          role: backend
      template:
        metadata:
          name: backend
          labels:
            app: gceme
            role: backend
        spec:
          containers:
          - name: backend
            image: <REPLACE_WITH_YOUR_OWN_IMAGE>
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
            imagePullPolicy: Always
            command: ["sh", "-c", "app -run-migrations -mode=backend -run-migrations -port=8080 -db-host=db -db-password=$MYSQL_ROOT_PASSWORD" ]
            ports:
            - name: backend
              containerPort: 8080
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: frontend
    spec:
      type: LoadBalancer
      ports:
      - name: http
        port: 80
      selector:
        app: gceme
        role: frontend
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gceme
          role: frontend
      template:
        metadata:
          name: frontend
          labels:
            app: gceme
            role: frontend
        spec:
          containers:
          - name: frontend
            image: <REPLACE_WITH_YOUR_OWN_IMAGE>
            imagePullPolicy: Always
            command: ["app", "-mode=frontend", "-backend-service=http://backend:8080", "-port=80"]
            ports:
            - name: frontend
              containerPort: 80
    ```

1. In another Cloud Shell Terminal, SSH to any of the nodes and examine the generated [iptables rules](http://ipset.netfilter.org/iptables.man.html)

    ```shell
    sudo iptables-save | grep backend
    ```

    The output should resemble this:

    ```
    -A KUBE-NODEPORTS -p tcp -m comment --comment "kube-system/default-http-backend:http" -m tcp --dport 31213 -j KUBE-MARK-MASQ
    -A KUBE-NODEPORTS -p tcp -m comment --comment "kube-system/default-http-backend:http" -m tcp --dport 31213 -j KUBE-SVC-XP4WJ6VSLGWALMW5
    -A KUBE-SEP-GPHKOMS2PXBGUJUI -s 10.16.0.8/32 -m comment --comment "kube-system/default-http-backend:http" -j KUBE-MARK-MASQ
    -A KUBE-SEP-GPHKOMS2PXBGUJUI -p tcp -m comment --comment "kube-system/default-http-backend:http" -m tcp -j DNAT --to-destination 10.16.0.8:8080
    -A KUBE-SERVICES ! -s 10.16.0.0/14 -d 10.19.247.83/32 -p tcp -m comment --comment "kube-system/default-http-backend:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
    -A KUBE-SERVICES -d 10.19.247.83/32 -p tcp -m comment --comment "kube-system/default-http-backend:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-XP4WJ6VSLGWALMW5
    -A KUBE-SVC-XP4WJ6VSLGWALMW5 -m comment --comment "kube-system/default-http-backend:http" -j KUBE-SEP-GPHKOMS2PXBGUJUI
    -A KUBE-SERVICES -d 10.19.249.235/32 -p tcp -m comment --comment "default/backend:http has no endpoints" -m tcp --dport 8080 -j REJECT --reject-with icmp-port-unreachable
    ```

## Modify a Service and Track iptables Changes

Redeploy the `backend` service in different configurations and observe the change to iptables. Make sure you understand the changes. Use `sudo iptables-save | grep backend` command to keep track of the relevant iptables rules

Try the following configurations:

1. Scale up the number of pods, covered by the service, to 3
1. Scale down the number of pods, covered by the service, to 1
1. Change service type to `NodePort`
1. Change service type to `LoadBalancer` (or examine the rules generated for the frontend service)

<details><summary>SOLUTIONS - CLICK ME</summary>
<p>

NodePort:

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: backend
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 30080
  selector:
    role: backend
    app: gceme
```

```shell
kubectl get nodes -o yaml | grep InternalIP -C 2
```

You can modify the `frontend` Deployment to use the Node internal IP and port 30080.

If you set up firewall rules, you will also be able to curl the backend using the external IP address (by default this is blocked).

```shell
curl https://<EXTERNAL-IP>:<NODE-PORT> -k
```

LoadBalancer:

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: backend
spec:
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  selector:
    role: backend
    app: gceme
```

You can modify the `frontend` Deployment to use the `backend` Service external IP and port 80.

> Note: As it takes some time to provision the load balancer on the infrastructure, you mus deploy the frontend separately once the external IP of the backend is available.

</p>
</details>

## Clean Up

```
kubectl delete -f manifests/sample-app.yaml
```

---

Next: [Istio](10-istio.md)
