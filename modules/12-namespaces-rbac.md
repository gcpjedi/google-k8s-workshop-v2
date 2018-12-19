# Namespaces RBAC and IAM

## Module Objectives

1. Create a Namespace
1. Add a user to the cluster
1. Create a Role, assign it to the user and make sure it is enforced
1. Create a ClusterRole, assign it to the user and make sure it is enforced

---

## Create a Namespace

Namespaces provide for a scope of Kubernetes objects. You can think of it as a workspace you are sharing with other users.

With namespaces one may have several virtual clusters backed by the same physical cluster. Names are unique within a Namespace, but not across Namespaces.

The cluster administrator can divide physical resources between Namespaces using quotas.

Namespaces cannot be nested.

Low-level infrastructure resources like Nodes and PersistentVolumes are not associated with a particular Namespace.

1. List all Namespaces in the system

    ```shell
    kubectl get ns
    ```

1. Use `describe` to learn more about a particular Namespace.

    ```shell
    kubectl describe ns default
    ```

1. Save the following file as `manifests/workshop-ns.yaml`.

    ```shell
      apiVersion: v1
      kind: Namespace
      metadata:
        name: workshop
    ```

1. Apply the manifest to create a new Namespace called `workshop`.

    ```shell
    kubectl apply -f manifests/workshop-ns.yaml
    ```

1. List Namespaces again.

    > Note: You should see namespace `workshop` in the list

## Add a User to the Cluster

Kubernetes has two types of users. The first is human users. They are managed outside the cluster in gSuite. The second type is service accounts. Service accounts are used by processes to access the Kubernetes API.

In this exercise you will create a service account and configure `gcloud` to use it.

>Note: Typically you login as a human user, not as a service account. You would normally create additional human users in gSuite. Unfortunately, adding human users is out of scope for this workshop.

1. Create a service account in the new workspace.

    ```shell
    kubectl create serviceaccount workshop-user --namespace workshop
    ```

1. Set up authentication for `kubectl` with the service account. You will use the context `gke-workshop` for normal operations and `limited` for operations as `workshop-user`.

    ```shell
    kubectl config set-credentials workshop-user --token=$(kubectl get secret $(kubectl get secret -n workshop | grep workshop-user-token | cut -f 1 -d " ") -n=workshop -o jsonpath={.data.token} | base64 --decode)
    ```

1. Edit `~/.kube/config`.

   1. Duplicate the context section, name the new context `limited`.

   1. Rename the original context `gke-workshop`.

   1. Add `namespace: workshop` On the second context.

   1. Change the user to `workshop-user` on the second context.

   1. Set `current-context: gke-workshop`.

   It should look like this:

    ```yaml
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
    current-context: gke-workshop
    ```

1. Check if you can get Pods with the `gke-workshop` context.

    ```shell
    kubectl auth can-i get pods
    ```

    > Note: Output should be yes

1. Switch between the contexts.

    ```shell
    kubectl config use-context limited
    ```

1. Check if you can get Pods with the `limited` context.

    ```shell
    kubectl auth can-i get pods
    ```

    > Note: Output should be no. The user can do nothing as you didn't associate it with any role

1. Switch back to the normal context.

    ```shell
    kubectl config use-context gke-workshop
    ```

## Create a Role, Assign it to the User and Make Sure it is Enforced

A Role is a set of rules that are applied to the namespace. ClusterRole is applied to the whole cluster.

In both cases you describe the objects you want to grant access to and operations user may execute against these objects.

For the Role to take effect you must bind it to the user.

1. Create `manifests/worker-role.yaml` file which grants permissions to create Pods and Deployments.

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
    - apiGroups: ["extensions", "apps"]
      resources: ["deployments"]
      verbs: ["get", "watch", "list", "create"]
    ```

    > Note: The nginx deployment created later might be of either `extensions` or `apps` API groups depending on your client version. For simplicity, we will use both in this workshop.

1. Apply the manifest.

    ```shell
    kubectl apply -f manifests/worker-role.yaml
    ```

1. Create a RoleBinding `manifests/worker-rolebinding.yaml` between user and the Role.

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

    > Note: This RoleBinding is for the `workshop` Namespace only.

1. Apply the manifest.

    ```shell
    kubectl apply -f manifests/worker-rolebinding.yaml
    ```

1. Switch context to `limited`.

    ```shell
    kubectl config use-context limited
    ```

1. Create an `nginx` Deployment.

    ```shell
    kubectl run nginx --image=nginx -n workshop
    ```

1. Check that the Deployment and Pod was created successfully.

    ```shell
    kubectl get pods -n workshop
    ```

    ```
    NAME                     READY   STATUS    RESTARTS   AGE
    nginx-64f497f8fd-kqgzr   1/1     Running   0          18s
    ```

1. Check that the user still can't read nodes in the cluster.

    ```shell
    kubectl get nodes
    ```

## Create a ClusterRole, Assign it to the User and Make Sure it is Enforced

1. Create a `ClusterRole` to list nodes `manifests/cr-node-reader.yaml`.

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

    > Note: Nodes are cluster-wide resources and are not associated with a particular Namespace.

1. Apply the manifest.

    ```shell
    kubectl apply -f manifests/cr-node-reader.yaml
    ```

1. Bind the `node-reader` ClusterRole to the service account `manifests/crb-node-reader.yaml`.

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

1. Apply the manifest.

    ```shell
    kubectl apply -f manifests/crb-node-reader.yaml
    ```

1. Now verify that user may list nodes.

    ```shell
    kubectl config use-context limited
    ```

    ```shell
    kubectl get nodes
    ```

    ```
    NAME                                          STATUS   ROLES    AGE   VERSION
    gke-gke-workshop-default-pool-5d910404-t38p   Ready    <none>   3h    v1.11.3-gke.18
    ```

1. Switch back to the unrestricted context.

    ```shell
    kubectl config use-context gke-workshop
    ```

## Optional Exercise

Delete `pods` permissions from the `worker` role and try to deploy `nginx`. What happens? Can you explain the outcomes?

---

Next: [Secure Container Images](13-secure-container-images.md)
