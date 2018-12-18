# Istio

## Module Objectives

1. Configuring & installing Istio
1. Deploying a microservice with an Istio sidecar
1. Monitoring and tracing
1. Traffic Shifting
1. Fault Injection
1. Circuit Breaking
1. Control Egress Traffic

---

## Configure & Install Istio

In this exercises you will create a new cluster with Istio installed on top of it. You will use an automation tool called `Cloud Deployment Manager` to accomplish the task. This tool allows one to describe infrastructure in the configuration file and manage it declaratively.

1. Grant cluster admin permissions to the current user:

    ```shell
    kubectl create clusterrolebinding cluster-admin-binding \
      --clusterrole=cluster-admin \
      --user="$(gcloud config get-value core/account)"
    ```

    > Note: You need these permissions to create the necessary role based access control (RBAC) rules for Istio

1. Download the Istio release

    ```shell
    cd $HOME
    wget https://github.com/istio/istio/releases/download/1.0.4/istio-1.0.4-linux.tar.gz
    tar xzf istio-1.0.4-linux.tar.gz
    cd istio-1.0.4
    ```

1. Add the `istioctl` client to your PATH. Add the following line to the `.bashrc` file:

    ```shell
    export PATH=$HOME/istio-1.0.4/bin:$PATH
    ```

1. Install Istio's core components:

    ```shell
    kubectl apply -f install/kubernetes/istio-demo-auth.yaml
    ```

    This does the following:

    * Creates the `istio-system` Namespace along with the required RBAC permissions

    * Deploys the core Istio components:

        * `Istio-Pilot` is responsible for service discovery and for configuring the Envoy sidecar proxies in an Istio service mesh

        * The Mixer components `Istio-Policy` and `Istio-Telemetry` enforce usage policies and gather telemetry data across the service mesh

        * `Istio-Ingressgateway` provides an ingress point for traffic from outside the cluster

        * `Istio-Citadel` automates key and certificate management for Istio

    * Deploys plugins for metrics, logs, and tracing

    * Enables mutual TLS authentication between Envoy sidecars

1. Verify that Istio is correctly installed

    ```shell
    kubectl get service -n istio-system
    kubectl get pods -n istio-system
    ```

    All pods should be in `Running` or `Completed` state.

Now you are ready to deploy the sample application to the Istio cluster.

## Deploying a microservice with an Istio sidecar

1. Change the `frontend` service type to `ClusterIP`. Refer to previous exercises for details

   > Note: you may need to delete the spec->ports[\*]->nodePort key, if it exists

1. Inject the sidecar container to the sample app from the previous section

    ```shell
    istioctl kube-inject -f manifests/sample-app.yaml  > manifests/sample-app-istio.yaml
    ```

    Inspect the generated file. Alternatively you can label a Namespace with `istio-injection=enabled` for automatic sidecar injection within the namespace.

1. Redeploy the sample app

    ```shell
    kubectl apply -f manifests/sample-app-istio.yaml
    ```

1. Create an Istio Gateway as `manifests/istio-gateway.yaml` and apply the chagnes

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

1. Create a VirtualService for the frontend as `manifests/frontend-vs.yaml` and apply the changes

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: frontend-vs
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

1. Check that the Gateway is created

    ```shell
    kubectl get gateway
    ```

    ```
    NAME            AGE
    gceme-gateway   3m
    ```

1. Get Ingress IP info (from the load balancer)

    ```shell
    export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
    export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
    export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
    ```

1. Now the app should be reachable through the Istio gateway on `$GATEWAY_URL`

You can write notes and save them in the database. But you don't see a majority of the information about the GCE instance. This is because the app gets this info from the `metadata.google.internal` server, which is not part of the Istio service mesh.

## Monitoring and Tracing

1. Set up a tunnel to Grafana

    ```
    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
    ```

1. Open the app in a web browser and send a couple of requests

1. Open web preview at port `3000`, you should see the Grafana interface

1. In the left panel click on `Dashboards -> Manage` and then select the `istio` folder. You should see a lot of dashboards, let's check a couple of them

    * `Istio Mesh Dashboard` - Displays the overall volume of the requests as well as number of failed requests and track requests latency. Refresh the frontend page several times, add a couple of notes and make sure you see a spike in global request volume graph and changes in overall request statistics.
    * `Istio Service Dashboard` - Contains more detail information about request statistics. You can use this dashboard to see the request statistics per service.
    * `Istio Workload Dashboard` - Gives details about metrics for each workload and then inbound workloads (workloads that are sending request to this workload) and outbound services (services to which this workload send requests) for that workload.

1. Set up a tunnel to ServiceGraph

    ```shell
    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8088:8088 &
    ```

1. Open web preview at port `8088` and append `dotviz` to the path. You should see sample-app's service topology

1. Setup access to the Jaeger dashboard by using port-forwarding

    ```shell
    kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686 &
    ```

1. Open web preview at port `16686`, you should see the Jaeger interface

1. Open the app and add some notes

1. In the Jaegger UI select `Service=gcme`, `Operation=frontend.default.svc.cluster.local:80/add-note` and click "Find Traces"

1. Make sure that in the Jaeger UI you see all requests that you've just sent. Open one of the requests. You should see all sub-requests that were send in the context of main request (including request to the backend and request to the Istio internal components)

## Traffic Shifting

Let's now see how Istio can help us to add new features to our application. Let's imagine that we want to add a new feature to the app and test it on a small percent of our users (this is called 'Canary deployment')

1. In the `sample-app` folder open `main.go` file.

1. At line 60 find `version` constant and change it from `1.0.0` to `1.0.1`

1. Now rebuild the app image with a new version tag and push it to the container registry. (These commands should be executed in the `sample-app` folder)

    ```shell
    export IMAGE_V1=gcr.io/$PROJECT_ID/sample-k8s-app:1.0.1
    docker build . -t $IMAGE_V1
    docker push $IMAGE_V1
    ```

1. Edit `manifests/sample-app.yaml`. Duplicate the `backend` deployment section. Keep the name for the first deployment (`backend`) and name the second one `backend-v1`. Modify the first deployment in the following way: add `version: v0` label to the `spec -> selectors -> matchLabels` and to the `spec -> templates -> metadata -> labels` elements. Do the same for the second deployment, but this time use `version: v1` label instead.

1. Change the second deployment image. Change the image tag from `1.0.0` to `1.0.1`

1. Configure the default namespace for automatic sidecar injection

    ```shell
    kubectl label namespace default istio-injection=enabled
    ```

1. Apply changes as usual (without `istioctl kube-inject`)
    ```shell
    kubectl apply -f manifests/sample-app.yaml
    ```

1. Create a destination rule for the backend service as `manifests/backend-dr.yaml` and apply the changes

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: backend-dr
    spec:
      host: backend
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
      subsets:
      - name: v0
        labels:
          version: v0
      - name: v1
        labels:
          version: v1
    ```

1. Create a VirtualService for the backend as `manifests/backend-vs.yaml` and apply the changes

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: backend-vs
    spec:
      hosts:
      - backend
      http:
      - route:
        - destination:
            host: backend
            subset: v0
          weight: 75
        - destination:
            host: backend
            subset: v1
          weight: 25
    ```

1. Open the app and refresh the page several times. You should see `1.0.0` backend version in 75% of cases and `1.0.1` in 25%

## Fault Injection

One of the most difficult aspects of testing microservice applications is verifying that the application is resilient to failures. Each service should not just assume that all its dependencies are available 100% of the time - instead it should be ready to handle any unexpected failure. Usually people manually shut down application instances or block application ports in order to simulate failures. Istio provides us with a much better way: fault injection.

1. Modify `manifests/backend-vs.yaml` and add the following lines to `spec -> http[0]` (if you simply append them to the end of the file it should work fine). Apply the changes

    ```
        fault:
          delay:
            fixedDelay: 3s
            percent: 50
    ```

1. Open the app and verify that in 50% of the times it should take 3 seconds to complete the request  

In a similar way you can inject not only delays, but also failures

## Retries and Circuit Breaking

Now let's inject a native failure to the backend application to demonstrate how Istio can help make microservices more resilient

1. Modify `manifests/sample-app.yaml` and add `-fail-percent=50` parameter to the `backend` deployment startup command (Leave `backend_v1` deployment untouched) Apply the changes

1. Delete the fault definition from the `manifests/backend-vs.yaml`. Apply the changes

1. Observe that the application is failing 50% of the times (Failures should happen only if frontend connects to the `1.0.0` version of the backend)

1. Add retries to the `spec -> http[0]` section of the `manifests/backend-vs.yaml`. Apply the changes

    ```
        retries:
          attempts: 3
          perTryTimeout: 2s
    ```
1. Test the app: you should no longer see any failure. Each failed request now retries up to 3 times

Now let's demonstrate how we can automatically remove a failing app from the system (apply Circuit Breaking pattern)

1. Delete the `retries` section from the `manifests/backend-vs.yaml` and apply the changes. Observe that the app starts failing again

1. Add the following lines to the `spec -> trafficPolicy` section of the `manifests/backend-dr.yaml` and apply the changes

    ```
        outlierDetection:
          consecutiveErrors: 1
          interval: 1s
          baseEjectionTime: 1m
          maxEjectionPercent: 100
    ```

1. Check that after the first error `backend` `1.0.0` is ejected and all requests are redirected to the `1.0.1` backend.

> Note: There is a known Istio [issue](https://github.com/istio/istio/issues/8846) that prevents this functionality from working correctly - ejected pods are reenabled right after they were ejected.

To verify that the backend pod was actually ejected do the following:

1. List all pods

1. Exec into the sidecar container of the frontend pod
    ```
    exec -it <frontend-pod> -c istio-proxy bash
    ```

1. Get relevant statistics
    ```
    curl localhost:15000/stats | grep outlier_detection
    ```

1. Check `ejections_enforced_total` parameter - it should be non zero.

## Control Egress Traffic

We still have one major issue with our app: it can't access external services and get GCP metadata. We can resolve this issue using a ServiceEntry

1. Save the following as `manifests/external.yaml` and apply the changes

    ```
    apiVersion: networking.istio.io/v1alpha3
    kind: ServiceEntry
    metadata:
      name: gcp-metadata
    spec:
      hosts:
      - 169.254.169.254 # ip address of the gcp metadata server
      ports:
      - number: 80
        name: http
        protocol: HTTP
      location: MESH_EXTERNAL
    ```

1. Check that the app can access GCP metadata

---

Next: [Audit Logging](11-audit-logging.md)
