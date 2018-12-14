# Namespaces RBAC and IAM

## Module Objectives

1. Create a namespace
1. Add a user to the cluster
1. Create role, assign it to the user and make sure it is enforced
1. Create cluster role, assign it to the user and make sure it is enforced

## Create a namespace

Namespaces provide for a scope of Kubernetes objects. You can think of it as a workspace you're sharing with other users.

With namespaces one may have several virtual clusters backed by the same physical cluster. Names are unique within a namespace, but not across namespaces.

Cluster administrator can divide physical resources between namespaces using quotas.

Namespaces cannot be nested.

Low-level infrastructure resources like Nodes and PersistentVolumes are not associated with a particular namespace

1. List all namespaces in the system.

    ```shell
    kubectl get ns
    ```

1. Use `describe` to learn more about a particular namespace.

    ```shell
    kubectl describe ns default
    ```

1. Create a new namespace called `workshop`

    Save the following file as `workshop-ns.yaml`

    ```shell
    cat > workshop-ns.yaml << EOM
      apiVersion: v1
      kind: Namespace
      metadata:
        name: workshop
    EOM
    ```

    Apply the manifest

    ```shell
    kubectl apply -f workshop-ns.yaml
    ```

1. List namespaces again.

    You should see namespace `workshop` in the list.

## Add a user to the cluster

Kubernetes has two types of users. First is human users. They are managed outside the cluster in gSuite. The second type is service accounts. Service accounts are used by processes to access Kubernetes API.

In this exercise you will create service account and configure `gcloud` to use it. Typically you login as a human user but to implement this scenario you would need to create a user in gSuite which is out of scope of this training.

1. Create service account in the new workspace

    ```shell
    $ kubectl create serviceaccount workshop-user --namespace workshop
    serviceaccount/workshop-user created
    ```

1. Set up authentication for `kubectl` with the service account. You will use context `gke-workshop` for noraml operations and `limited` for operations as `workshop-user`.

    ```shell
    $ kubectl config set-credentials workshop-user --token=$(kubectl get secret workshop-user-token-mv6d9 -n=workshop -o jsonpath={.data.token} | base64 --decode)

    # edit ~/.kube/config
    contexts:
    - context:
        cluster: gke_project-aleksey-zalesov_europe-west1-d_gke-workshop
        user: gke_project-aleksey-zalesov_europe-west1-d_gke-workshop
      name: gke-workshop
    - context:
        cluster: gke_project-aleksey-zalesov_europe-west1-d_gke-workshop
        namespace: workshop
        user: workshop-user
      name: limited

    # switch between the contexts
    $ kubectl auth can-i get pods
    yes

    $ kubectl config use-context limited
    Switched to context "limited".

    $ kubectl auth can-i get pods
    no
    ```

    The user can do nothing as you didn't associate it with any role.

1. Switch back to the normal account

    ```shell
    $ kubectl config use-context gke-workshop
    Switched to context "gke-workshop".
    ```

## Create role, assign it to the user and make sure it is enforced

Role is a set of rules that are applied to the namespace. ClusterRole is applied to the whole cluster.

In both cases you describe the objects you want to grant access to and operations user may execute against these objects.

For the role to take effect you must bing it to the user.

1. Create `worker-role.yaml` file which grants permissions to create pods and deployments.

    ```yaml
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      namespace: workshop
      name: worker
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "watch", "list", "create"]
    - apiGroups: ["apps"]
      resources: ["deployments"]
      verbs: ["get", "watch", "list", "create"]
    ```

    Apply the manifest

    ```shell
    $ kubectl apply -f worker-role.yaml
    role.rbac.authorization.k8s.io/worker created
    ```

1. Create binding between user and the role. Note that it is for `workshop` namespace only.

    ```yaml
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: worker
      namespace: workshop
    subjects:
    - kind: User
      name: system:serviceaccount:workshop:workshop-user
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role
      name: worker
      apiGroup: rbac.authorization.k8s.io
    ```

    Apply the manifest.

1. Switch context to `limited` and `nginx`

    ```shell
    $ kubectl config use-context limited
    Switched to context "limited".

    $ kubectl run nginx --image=nginx
    deployment.apps/nginx created

    $ kubectl get pods
    NAME                     READY   STATUS    RESTARTS   AGE
    nginx-64f497f8fd-kqgzr   1/1     Running   0          18s
    ```

1. The user still can't read nodes in the cluster

    ```shell
    kubectl get nodes
    ```

## Create cluster role, assign it to the user and make sure it is enforced

1. Create `ClusterRole` to list nodes. Nodes are cluster-wide resources and are not associated with a particular namespace.

    ```yaml
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: node-reader
    rules:
    - apiGroups: [""]
      resources: ["nodes"]
      verbs: ["get", "watch", "list"]
    ```

    Apply the manifest.

1. Bind `node-reader` role to the service account.

    ```yaml
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: read-nodes
    subjects:
    - kind: User
      name: system:serviceaccount:workshop:workshop-user
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: node-reader
      apiGroup: rbac.authorization.k8s.io
    ```

    Apply the manifest.

1. Now verify that user may list nodes.

    ```shell
    $ kubectl config use-context limited
    Switched to context "limited".

    $ kubectl get nodes
    NAME                                          STATUS   ROLES    AGE   VERSION
    gke-gke-workshop-default-pool-5d910404-t38p   Ready    <none>   3h    v1.11.3-gke.18
    ```

1. Switch back to the unrestricted context

## Optional exercise

Delete `pods` permissions from the `worker` role and try to deploy `nginx`. What happens? Can you explain the outcomes?
