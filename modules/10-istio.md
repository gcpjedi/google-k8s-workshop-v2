# Istio

## Module Objectives

1. Configuring & installing Istio
1. Deploying a microservice with an istio sidecar
1. Monitoring and tracing
1. Route Rules and Virtual Services
1. Request Routing
1. Fault Injection
1. Traffic Mirroring
1. Circuit Breaking
1. Controlling ingress traffic [Using Istio Ingress]
1. Traffic Shifting
1. Rate-limiting using Istio & Memorystore [Redis]"

---

## Configure & Install Istio

In this exercises you will create a new cluster with Istio installed on top of it. You will use automation tool called `Cloud Deployment Manager` to accomplish the task. This tool allows one to describe infrastructure in the configuration file and manage it declaratively.

1. Grant cluster admin permissions to the current user. You need these permissions to create the necessary role based access control (RBAC) rules for Istio:

    ```shell
    kubectl create clusterrolebinding cluster-admin-binding \
      --clusterrole=cluster-admin \
      --user="$(gcloud config get-value core/account)"
    ```

1. Download Istio release

    ```shell
    wget https://github.com/istio/istio/releases/download/1.0.4/istio-1.0.4-linux.tar.gz
    tar xzf istio-1.0.4-linux.tar.gz
    cd istio-1.0.4
    ```

1. Add the istioctl client to your PATH. Add the following line to the `.bashrc` file

    ```shell
    export PATH=$HOME/istio-1.0.4/bin:$PATH
    ```

1. Install Istio's core components:

    ```shell
    kubectl apply -f install/kubernetes/istio-demo-auth.yaml
    ```

    This does the following:

    * Creates the istio-system namespace along with the required RBAC permissions
    * Deploys the core Istio components:

        * Istio-Pilot, which is responsible for service discovery and for configuring the Envoy sidecar proxies in an Istio service mesh.
        * The Mixer components Istio-Policy and Istio-Telemetry, which enforce usage policies and gather telemetry data across the service mesh.
        * Istio-Ingressgateway, which provides an ingress point for traffic from outside the cluster.
        * Istio-Citadel, which automates key and certificate management for Istio.
    * Deploys plugins for metrics, logs, and tracing.

    * Enables mutual TLS authentication between Envoy sidecars.

1. Verify that Istio is correctly installed

    ```shell
    kubectl get service -n istio-system
    kubectl get pods -n istio-system
    ```

    All pods should be in Running or Completed state.

Now you are ready to deploy sample application to the Istio cluster.

## Deploying a microservice with an istio sidecar

1. Change `frontend` service type to Cluster IP.

1. Inject sidecar container to the sample app
    ```
    istioctl kube-inject -f sample-app.yml  > sample-app-istio.yml
    ```
    Inspect the generated file. Alternatively you can label a namespace for automatic sidecar injection by assigning the `istio-injection=enabled` label to the namespace.

1. Redeploy the sample app

    ```
    kubectl apply -f sample-app-istio.yml
    ```

1. Create Istio gateway

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: gceme-gateway
    spec:
      selector:
        istio: ingressgateway # use istio default controller
      servers:
      - port:
          number: 80
          name: http
          protocol: HTTP
        hosts:
          - "*"
    ```

1. Create virtual service for the frontend

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: gceme
    spec:
      hosts:
      - "*"
      gateways:
      - gceme-gateway
      http:
      - match:
        - uri:
            exact: /
        - uri:
            exact: /add-note
        - uri:
            exact: /healthz
        route:
        - destination:
            host: frontend
            port:
              number: 80
    ```

1. Check gateway is created

    ```shell
    $ kubectl get gateway -n dev
    NAME            AGE
    gceme-gateway   3m
    ```

1. Get Ingress IP info (from load balancer)

    ```shell
    export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
    export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
    export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
    ```


1. Now the app should be reachable through the Istio gateway on `$GATEWAY_URL`

You can write notes and save them in the database. But you don't see majority of the information about GCE instance. This is because application gets this info from the `metadata.google.internal` server which is not part of the Istio mesh. We will learn how to exclude some IPs from the Istio policy later and for now let' proceed to the next exercise.

## Monitoring and tracing


1. Set up a tunnel to Grafana.
    ```
    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
    ```
1. Open the app in a web browser and send a couple of requests.

1. Open web preview at port `3000`- you should see grafana interface

1. In the left panel click on `Dashboards -> Manage` and then select `istio` folder. You should see a lot of dashboards, let's check a couple of them.

    * `Istio Mesh Dashboard` - this dashboard allows you to see the overal volume of the requests as well as number of failed requests and track requests latency. Refresh the frontend page several times, add a couple of notes and make sure you see a apike in global request volue graph and changes in overal request statistics.
    * `Istio Service Dashboard` - more detail information about request statistics. You can use this dashboard to see the request statistics per service.
    * `Istio Workload Dashboard` - this gives details about metrics for each workload and then inbound workloads (workloads that are sending request to this workload) and outbound services (services to which this workload send requests) for that workload.

1. Set up a tunnel to ServiceGraph 

    ```
    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8088:8088 &
    ```

1. Open web preview at port `8088` and append `dotviz` to the path. You should see sample-app service topology.

---

Next: [Audit Logging](11-audit-logging.md)
