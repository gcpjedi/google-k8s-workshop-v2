# Istio

## Module objectives

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

## Configure & Install Istio

In this exercises you will create a new cluster with Istio installed on top of it. You will use automation tool called `Cloud Deployment Manager` to accomplish the task. This tool allows one to describe infrastructure in the configuration file and manage it declaratively.

Google Cloud Marketplace is a registry of such automation templates you may use to bootstrap services on top of GCP.

1. Download Istio release

    ```shell
    wget https://github.com/istio/istio/releases/download/1.0.4/istio-1.0.4-linux.tar.gz
    tar xzf istio-1.0.4-linux.tar.gz
    cd istio-1.0.4
    ``

1. Create Tiller service account

    ```shell
    kubectl apply -f install/kubernetes/helm/helm-service-account.yaml
    ```

1. Install Tiller

    ```shell
    helm init --service-account tiller
    ```

1. Install Istio

    ```shell
    helm install install/kubernetes/helm/istio \
      --name istio \
      --namespace istio-system \
      --values install/kubernetes/helm/istio/values-istio-demo-auth.yaml
    ```

1. Verify that Istio is correctly installed

    ```shell
    kubectl get service -n istio-system
    kubectl get pods -n istio-system
    ```

    All pods should be in Running or Completed state.

Now you are ready to deploy sample application to the Istio cluster.

## Deploying a microservice with an istio sidecar

1. Create namespace called `dev`

1. Deploy `sample-app` to the `dev` namespace

    You should be able to access the LoadBalancer IP as usual.

1. Configure injection of the sidecar into the app

    ```shell
    # label default namespace for sidecar injection
    $ kubectl label namespace dev istio-injection=enabled

    #recreate containers in the dev namespace

    # check they are now started with the sidecar
    $ k get pods -n dev --watch
    NAME       READY   STATUS    RESTARTS   AGE
    backend    2/2     Running   2          61s
    db         2/2     Running   0          61s
    frontend   2/2     Running   0          61s
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

    ```shell
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
$ k get gateway -n dev
NAME            AGE
gceme-gateway   3m
```

1. Get Ingress IP info

```shell
# get data from LB
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

1. Change `frontend` service type to Cluster IP.

    Now the app should be reachable through the Istio gateway on `$GATEWAY_URL`

You can write notes and save them in the database. But you don't see majority of the information about GCE instance. This is because application gets this info from the `metadata.google.internal` server which is not part of the Istio mesh. We will learn how to exclude some IPs from the Istio policy later and for now let' proceed to the next exercise.
