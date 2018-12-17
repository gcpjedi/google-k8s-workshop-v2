# Ingress

## Module Objectives

1. Serve app traffic from the ingress instead of LoadBalancer service
1. Use static IP with Ingress
1. Specify app domain
1. Add SSL support

---

## Theory

Ingress is an API object that manages external access to the services in a cluster, typically HTTP.

Ingress can provide load balancing, SSL termination and name-based virtual hosting.

---

## Use Ingress

1. Change the frontend service type from LoadBalancer to NodePort.

    ```shell
    kubectl edit svc/frontend
    ```

    * Find the line `type: LoadBalancer` and change it to `type: NodePort`.

    * Save the file `Esc :wq`.

    * Ingress forwards traffic to a Service (not directly to Pods). That's why we still need a Service wrapping our frontend Pod. However, it is not necessary for this Service to be of a LoadBalancer type, because now we will be accessing the frontend through Ingress and not through a dedicated frontend load balancer. The Service has to be of a NodePort type instead.

1. Check the Service type.

    ```shell
    kubectl get svc
    ```

    ```
    NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
    backend      ClusterIP   10.111.2.72    <none>        8080/TCP       5h
    db           ClusterIP   10.111.13.61   <none>        3306/TCP       1d
    frontend     NodePort    10.111.10.20   <none>        80:31661/TCP   36m
    kubernetes   ClusterIP   10.111.0.1     <none>        443/TCP        1d
    ```

1. Create file `manifests/ingress.yaml`.

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: gceme-ingress
    spec:
      rules:
      - http:
          paths:
          - path: /gceme/*
            backend:
              serviceName: frontend
              servicePort: 80
    ```

    This will expose the service `frontend` using relative path `/gceme`.

1. Create the Ingress

    ```shell
    kubectl get ingress --watch  # in the first terminal
    ```

    ```shell
    kubectl apply -f manifests/ingress.yaml  # in the second terminal
    ```

    ```
    NAME            HOSTS   ADDRESS          PORTS   AGE
    gceme-ingress   *                        80      0s
    gceme-ingress   *       35.227.223.114   80      6m22s
    ```

    Wait until you see IP in the address field. The application will be available as `http://<ingress-ip>/gceme/`.

1. In the Cloud Console go to 'Network services' -> 'Load balancing' and examine the created load balancer.

## Use Static IP

By default, Ingress uses ephemeral IP which may change during the time. To create DNS record and issue SSL certificates one needs static IP. In this exercise, you will create one and use it with Ingress.

1. Create a static IP.

    ```shell
    gcloud compute addresses create web-static-ip --global
    ```

    > Note: You can retrieve the static IP using the gcloud CLI

    ```shell
    gcloud compute addresses list
    ```

1. Assign it to the Ingress.

    ```shell
    kubectl edit ingress/gceme-ingress
    ```

    ```yaml
    metadata:
      annotations:
        kubernetes.io/ingress.global-static-ip-name: "web-static-ip"
    ```

    When you save the file Kubernetes will change the IP of the load balancer according to the annotation. You may get new IP from the Cloud Console or Ingress resource.

## Optional Exercises

### Specify app domain

1. Now, the `gceme` app should be accessed using a specific DNS name.
1. Modify your `/etc/hosts` and set `gceme-training.com` domain to be resolved to the ingress IP address.
1. Modify the Ingress definition appropriately. Find the section `Name-based virtual hosting` in [this](https://kubernetes.io/docs/concepts/services-networking/ingress/#name-based-virtual-hosting) document for reference.
1. Access `gceme-training.com` from your web browser.
1. Verify that you can't access `gceme` app using IP address anymore.

<details><summary>SOLUTION - CLICK ME</summary>
<p>

1. The `spec` rules section should look like this.

    ```yaml
    rules:
    - host: gceme-training.com
      http:
        paths:
        - backend:
            serviceName: frontend
            servicePort: 80
          path: /gceme/*
    ```

    > Note: `/etc/hosts` should be modified on your local machine, not the Cloud Console.

</p>
</details>

### Use TLS

1. Create a self-signed certificate for `gceme` app for the `gceme-training.com` domain [link](https://stackoverflow.com/questions/10175812/how-to-create-a-self-signed-certificate-with-openssl).
1. Create a Kubernetes Secret for `gceme` app. The secret should contain the certificate and private key.
1. Add a `tls` section to the ingress definition. You can use the `tls` section from [this](https://kubernetes.io/docs/concepts/services-networking/ingress/#types-of-ingress) document for reference.
1. Redeploy, open app in a web browser and examine certificate details. Use [this](https://www.ssl2buy.com/wiki/how-to-view-ssl-certificate-details-on-chrome-56) link to see how a certificate can be viewed in chrome.

<details><summary>SOLUTION - CLICK ME</summary>
<p>

1. Create a self-signed certificate.

    ```shell
    openssl req -nodes -x509 -newkey rsa:2048 -keyout gceme_key.pem -out gceme_cert.pem -days 365 -subj "/C=US/ST=California/L=Sunnyvale/O=Altoros/OU=Training/CN=gceme-training.com"
    ```

1. Import the secret to Kubernetes.

    ```shell
    kubectl create secret tls gceme-tls --cert=gceme_cert.pem --key=gceme_key.pem
    ```

1. Add the tls certificates to the Ingress spec.

    ```yaml
    tls:
    - hosts:
      - gceme-training.com
      secretName: gceme-tls
    ```

1. The final `manifests/ingress.yaml` should look like the following:

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      annotations:
        kubernetes.io/ingress.global-static-ip-name: web-static-ip
      name: gceme-ingress
    spec:
      rules:
      - host: gceme-training.com
        http:
          paths:
          - backend:
              serviceName: frontend
              servicePort: 80
            path: /gceme/*
      tls:
      - hosts:
        - gceme-training.com
        secretName: gceme-tls
    ```

* You can troubleshoot certificate issues by viewing the Ingress events.

    ```shell
    kubectl describe ing gceme-ingress
    ```

</p>
</details>

---

Next: [CNI Networking](08-cni-networking.md)
