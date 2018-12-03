Istio
=====

Configure & Install Istio
-------------------------

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
