# Securing Network and Container Runtime

## Module Objectives

1. Create and test a Network Security Policy
1. Create and test a Pod Security Policy
1. Enable Istio mutual TLS
1. Use Istio Whitelist/Blacklist
1. Setup a GKE Private Cluster, make sure nodes don't have public IPs
1. Use Master Authorized Networks to secure access to the cluster master endpoint
1. Enable Metadata Concealment to prevent Pods from accessing certain VM metadata
1. Enable and test Cloud IAP for the cluster

---

## Create and Test a Network Security Policy

First let's enable network policy enforcement on the GKE cluster. It is a two-step process.

1. Deploy network policy add-on on top of the cluster

    ```shell
    gcloud container clusters update gke-workshop \
      --update-addons=NetworkPolicy=ENABLED
    ```

    It will take about 6 minutes so be patient, grab a cup of coffee

1. Enable Network Policy enforcement on the Nodes

    ```shell
    gcloud container clusters update gke-workshop \
      --enable-network-policy
    ```

    The rolling update takes 2 minutes per node, so the cluster consisting of four nodes will be upgraded in 8 minutes

1. Watch how your Kubernetes cluster is updated one Node at a time

    ```shell
    kubectl get nodes --watch
    ```

    When all the nodes are in `Ready` state you may proceed to the next step

Let's see how to use a Network Policy for blocking the external traffic for a `Pod`

1. Create a file `manifests/deny-egress.yaml`

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: foo-deny-egress
    spec:
      podSelector:
        matchLabels:
          app: foo
      policyTypes:
      - Egress
      egress:
      # Allow DNS resolution
      - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    ```

    ```shell
    kubectl apply -f manifests/deny-egress.yaml
    ```

    > Note: This network policy blocks all the outgoing traffic except DNS resolution

1. Now start the Pod that matches label `app=foo`

    ```shell
    kubectl run --rm --restart=Never --image=alpine -i -t --labels app=foo test -- ash
    ```

1. Test DNS resolution
  
    ```
    nslookup www.example.com
    ```
    ```
    nslookup: can't resolve '(null)': Name does not resolve

    Name:      www.example.com
    Address 1: 93.184.216.34
    Address 2: 2606:2800:220:1:248:1893:25c8:1946
    ```

1. Test HTTP traffic.  
  
    ```
    wget --timeout 1 -O- http://www.example.com
    ```   
    ```
    Connecting to www.example.com (93.184.216.34:80)
    wget: download timed out
    ```

    You see the name resolution works fine but external connections are dropped

    `nslookup: can't resolve '(null)': Name does not resolve` is a [bug](https://forums.docker.com/t/resolved-service-name-resolution-broken-on-alpine-and-docker-1-11-1-cs1/19307/11), ignore it for now


## Create and Test a Pod Security Policy

To enable PSP for the new cluster, use `--enable-pod-security-policy` flag during creation

We have already created the cluster so we will use the update command

1. Update the cluster to enable Pod Security Policies (PSP)

    ```shell
    gcloud beta container clusters update gke-workshop \
      --enable-pod-security-policy
    ```

    > Note: that you must use `beta` version of `gcloud` to execute this command

    It will take about 6 minutes for updating the cluster, during which you won't be able to access API

1. Start from the PSP boilerplate and add security measures to disable privilege escalation to root.

    Create the file `manifests/psp.yaml`

    ```yaml
    apiVersion: policy/v1beta1
    kind: PodSecurityPolicy
    metadata:
      name: restricted
    spec:
      privileged: false
      allowPrivilegeEscalation: false
      seLinux:
        rule: 'RunAsAny'
      runAsUser:
        rule: 'MustRunAsNonRoot'
      supplementalGroups:
        rule: 'MustRunAs'
        ranges:
          - min: 1
            max: 65535
      fsGroup:
        rule: 'MustRunAs'
        ranges:
          - min: 1
            max: 65535
    ```

    ```shell
    kubectl apply -f manifests/psp.yaml
    ```

1. Typically `Pods` are controlled by `Deployments` and `ReplicaSets`. These controllers act on behalf of the `default` service account. We need to grant `use` permissions to the `default` service account.

    Create the file `manifests/pod-starter-role.yaml`

    ```yaml
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: pod-starter
    rules:
    - apiGroups:
      - extensions
      resources:
      - podsecuritypolicies
      resourceNames:
      - restricted
      verbs:
      - use
    ```

    ```shell
    kubectl apply -f manifests/pod-starter-role.yaml
    ```

1. Bind the ClusterRole to the `default` service account

    Create the file `manifests/psp-binding.yaml`

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: RoleBinding
    metadata:
      name: pod-starter-binding
      namespace: default
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: pod-starter
    subjects:
    - kind: ServiceAccount
      name: default
      namespace: default
    ```

    ```shell
    kubectl apply -f manifests/psp-binding.yaml
    ```

1. Now try to create a privileged container

    ```console
    $ kubectl create -f- <<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-privileged
      labels:
        app: privileged
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: privileged
      template:
        metadata:
          labels:
            app: privileged
        spec:
          containers:
            - name:  nginx
              image: nginx
              securityContext:
                privileged: true
    EOF
    ```

    The `Deployment` creates a `ReplicaSet` which in turn creates a `Pod`

1. Show the `ReplicaSet` state

    ```shell
    kubectl get rs -l=app=privileged
    ```
    
    ```
    NAME                    DESIRED   CURRENT   READY     AGE
    privileged-6c96db7488   1         0         0         5m
    ```

    No pods created. Why?

1. Get events from the `ReplicaSet`

    ```shell
    kubectl describe rs -l=app=privileged
    ```
    
    ```
    Error creating: pods "privileged-6c96db7488-" is forbidden: unable to validate against any pod security policy: [spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]
    ```

    Admission controller forbids creating privileged containers as the applied policy states

1. Create the pod directly

    ```
    kubectl create -f- <<EOF
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-privileged
    spec:
      containers:
        - name:  nginx
          image: nginx
          securityContext:
            privileged: true
    EOF
    ```

    Can you explain the result?

## Setup a GKE Private Cluster

GKE cluster endpoints by default have public IPs. That means users outside may access both master API and SSH to the nodes.

A GKE private cluster is a method to deploy a cluster with private IPs only and restrict outside access. The method uses VPC peering to connect masters with nodes. Nodes have no Internet access. They use Private Google Access to access GCP services like Container Registry.

To interact with the cluster, you will need to provision a VM inside the cluster network or setup master authorized networks.

1. Create a private Kubernetes cluster on GKE

    ```shell
    gcloud container clusters create gke-workshop-priv \
        --create-subnetwork name=workshop-subnet \
        --enable-master-authorized-networks \
        --enable-ip-alias \
        --enable-private-nodes \
        --enable-private-endpoint \
        --master-ipv4-cidr 172.16.0.32/28 \
        --no-enable-basic-auth \
        --no-issue-client-certificate \
        --cluster-version 1.11.3 \
        --num-nodes 1 \
        --machine-type n1-standard-2
    ```

    * `--enable-private-nodes` configures Nodes to use internal IP addresses
    * `--enable-private-endpoint` disables the public master API endpoint
    * `--enable-ip-alias` is required to enable VPC peering between master and nodes
    * `--enable-master-authorized-networks` configures the cluster to allow connections from other subnets than the subnet the cluster uses

1. Try to connect to the cluster from your laptop

    `kubectl` hangs as private endpoints are not available over the internet

1. Create a jumpbox in the same subnet as the Kuberentes cluster

    ```shell
    gcloud compute instances create jumpbox-inner \
      --subnet=workshop-subnet \
      --machine-type=f1-micro \
      --scopes=cloud-platform
    ```

    `--scopes=cloud-platform` option allows `gcloud` SDK to download cluster credentials from within the instance

    `--subnet=workshop-subnet` creates the instance in the same subnet as our private cluster

    Connect to the jumpbox
    ```shell
    gcloud compute ssh jumpbox-inner
    ```

    The jumpbox has both public and private IPs. You can SSH to the jumpbox with its public interface and then connect to the Kubernetes cluster using private IPs.

1. Verify that you can access Kubernetes from the jumpbox

    The jumpbox has both public and private IPs. You SSH to the jumpbox with its public interface and connect to the Kubernetes cluster using private IP.
    ```shell
    gcloud container clusters get-credentials gke-workshop-priv --zone=us-west2-b
    ```

1. Verify that you can access Kubernetes from the jumpbox
    ```
    Fetching cluster endpoint and auth data.
    kubeconfig entry generated for gke-workshop-priv.
    ```

    ```shell
    gcloud container clusters get-credentials gke-workshop-1 --zone=europe-west1-d
    ```

    ```
    ```

    ```shell
    sudo apt-get update && sudo apt-get install kubectl
    ```

    ```shell
    kubectl get nodes -o wide
    ```

    ```
    NAME                                            STATUS   ROLES    AGE   VERSION          INTERNAL-IP   EXTERNAL-IP
      OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
    gke-gke-workshop-priv-default-pool-f80a37f2-wxfj   Ready    <none>   16m   v1.11.3-gke.18   10.70.0.2 Container-Optimized OS from Google   4.14.65+         docker://17.3.2
    ```

    ```shell
    kubectl cluster-info
    ```

    ```
    Kubernetes master is running at https://172.16.0.34
    GLBCDefaultBackend is running at https://172.16.0.34/api/v1/namespaces/kube-system/services/default-http-backend:ht
    tp/proxy
    Heapster is running at https://172.16.0.34/api/v1/namespaces/kube-system/services/heapster/proxy
    KubeDNS is running at https://172.16.0.34/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    Metrics-server is running at https://172.16.0.34/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
    ```

1. Now create the VM outside the cluster subnet

    ```shell
    gcloud compute instances create jumpbox-outer \
      --machine-type=f1-micro \
      --scopes=cloud-platform
    ```

    ```
    Created [https://www.googleapis.com/compute/v1/projects/project-aleksey-zalesov/zones/europe-west1-d/instances/jumpbox-outer].
    NAME           ZONE            MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
    jumpbox-outer  europe-west1-d  f1-micro                   10.132.0.2   35.205.132.74  RUNNING
    ```

1. SSH to `jumpbox-outer`
    ```shell
    gcloud compute ssh jumpbox-outer
    ```

1. Can you connect to the Kubernetes cluster from `jumpbox-outer`? Try it

1. To enable connections from the VM outside cluster subnet, add the machines private IP address to the list of masters authorized networks

    ```shell
    gcloud container clusters update gke-workshop-priv \
        --enable-master-authorized-networks \
        --master-authorized-networks <internal-ip-of-jumpbox-outer>/32
    ```

    > Note: This time you should be able to access Kubernetes API from `jumpbox-outer`

### Clean Up

Delete the provisioned resources.

```shell
gcloud compute instances delete jumpbox-inner jumpbox-outer
gcloud container clusters delete gke-workshop-priv
```

## Enable Metadata Concealment to Prevent Pods from Accessing Certain VM Metadata

Nodes running in Google Cloud may learn information about themselves by querying metadata server. It is accessible without authentication from any instance on the host `http://metadata/computeMetadata/v1/`.

Some metadata is considered sensitive, in particular instance identity token. Applications connecting to the instance may verify the instance identity with this token. A Pod may interact as instance if it gets access to the token and recieves the connection from an app.

1. Get the token from within the unrestricted cluster

```shell
kubectl run --rm --restart=Never --image=alpine -i -t test -- ash
```
```shell
apk add curl
curl -H "Metadata-Flavor: Google" 'http://metadata/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://www.example.com'
```

```
eyJhbGciOi...
```

You can get the instance identity token from the `Pod`

Let's disable this behavior

1. Create a new cluster

    ```shell
    gcloud beta container clusters create gke-workshop-0 \
    --cluster-version 1.11.3 \
    --num-nodes 1 \
    --machine-type n1-standard-1 \
    --workload-metadata-from-node=SECURE
    ```

1. Run a container with the shell inside the cluster

    ```shell
    kubectl run --rm --restart=Never --image=alpine -i -t test -- ash
    apk add curl
    curl -H "Metadata-Flavor: Google" 'http://metadata/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://www.example.com'
    ```

    ```
    This metadata endpoint is concealed.
    ```

    GKE runs proxy between Pods and Metadata server. This proxy conceals the identity token endpoint and `kube-env`

1. What if user wants to get the project ID where the Kubernetes cluster is running?

    ```shell
    curl -H "Metadata-Flavor: Google" 'http://metadata/computeMetadata/v1/project/project-id'
    ```

    > Note: It works as proxy does not conceal all the endpoints.

    In current configuration one may use the legacy endpoint `v1beta1` instead of `v1`. The `v1beta1` is considered less secure as it does not implement some safeguards. For example, it does not require the `Metadata-Flavor: Google` header which protects users from querying the endpoint from the insecure environment by accident.

    ```shell
    url 'http://metadata/computeMetadata/v1beta1/project/project-id'
    ```

1. Unfortunately, you may not disable legacy endpoints for the existing cluster, so lets rebuild the cluster

    ```shell
    gcloud beta container clusters create gke-workshop-1 \
    --cluster-version 1.11.3 \
    --num-nodes 1 \
    --machine-type n1-standard-1 \
    --workload-metadata-from-node=SECURE \
    --metadata disable-legacy-endpoints=true
    ```

1. After the cluster is ready and you run a shell inside of it, try to use the `v1beta` endpoint

    ```shell
    curl 'http://metadata/computeMetadata/v1beta1/project/project-id'
    ```

    ```
    Your client does not have permission to get URL <code>/computeMetadata/v1beta1/project/project-id</code> from this server. Legacy metadata endpoint accessed: /computeMetadata/v1beta1/project/project-id
    Legacy metadata endpoints are disabled. Please use the /v1/ endpoint.
    ```

Metadata concealment is a temporary security solution made available to users of GKE while the bootstrapping process for cluster nodes is being redesigned with significant security improvements. This feature is scheduled to be deprecated and removed in the future.

### Clean Up

Get rid of both clusters

```shell
gcloud container clusters delete gke-workshop-0 --async
gcloud container clusters delete gke-workshop-1 --async
```

## Cloud IAP

1. Make sure that you have the `sample-app` exposed with an Ingress

    ```shell
    kubectl get ingress
    ```

    ```
    NAME            HOSTS   ADDRESS          PORTS     AGE
    basic-ingress   *       35.244.247.163   80, 443   34m
    ```

    Remember the external IP of the ingress - `35.244.247.163`

1. Cloud IAP requires that users connect using the HTTPS protocol. So now you will generate self-signed TLS certificate and add it to the Ingress.

    Generate a self-signed certificate

    ```shell
    openssl req \
      -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout tls.key \
      -out tls.crt \
      -subj "/CN=35.244.247.163.xip.io/O=nginxsvc"
    ```

    > Note: We use the xip.io domain as Cloud IAP doesn't allow using IP

1. Create a Secret with the tls certificate and private key

    ```shell
    kubectl create secret tls tls-secret-0 --key tls.key --cert tls.crt
    ```

1. Add the Secret to the Ingress

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: basic-ingress
    spec:
      tls:
      - secretName: tls-secret-0
    ...
    ```

1. Apply the configuration. Wait for the application to become available using HTTPS.

1. Add the consent screen to the project. This information will be displayed to the users accessing the application

    - URL: https://console.cloud.google.com/apis/credentials/consent
    - Application type: Internal
    - Application name: sample app
    - Authorized domains: xip.io
    - Application Homepage link: http://35.244.247.163.xip.io/home
    - Application Privacy Policy link: http://35.244.247.163.xip.io/privacy
    - Application Terms of Service link: http://35.244.247.163.xip.io/tos

    Click save

1. Add yourself as a user who can access the application

    Go to [Security IAP](https://console.cloud.google.com/security/iap)

    Select the Ingress for the sample application

    In the right pane click "Add member" and add yourself with the role `IAP-secured Web App User`

    Note that you can't add users outside the `altoros.com` organization as the application marked as `Internal`

    ![IAP Role](img/IAP-role.png)

1. Click the slider in the center of the screen to turn IAP on

1. Wait until Cloud IAP provisions credentials

    They will be available at https://console.cloud.google.com/apis/credentials

    ![IAP Credentials](img/iap-credentials.png)

1. When available, export as environmental variables:

    ```shell
    export CLIENT_ID="24..9oggcoseuc7q.apps.googleusercontent.com"
    export CLIENT_SECRET="NOTD.."
    ```

1. Create a Secret with OAuth data

    ```shell
    kubectl create secret generic my-secret-0 \
      --from-literal=client_id=$CLIENT_ID \
      --from-literal=client_secret=$CLIENT_SECRET
    ```

1. Use this secret as the backend configuration for Cloud IAP

    ```yaml
    apiVersion: cloud.google.com/v1beta1
    kind: BackendConfig
    metadata:
      name: config-default
      namespace: default
    spec:
      iap:
        enabled: true
        oauthclientCredentials:
          secretName: my-secret-0
    ```

1. Now wait some time and go to https://35.244.247.163.xip.io/

    > Note: The IP will be different in your case.

    You should see the screen `Sign in with Google`.

    Enter the account and the password you for the training and you will see the application frontend.

    Try to login with a different user and make sure you can't access the application.

You have successfully configured Cloud IAP to protect the application running on GKE.

---

[Return to Home](../README.md)
