# Namespaces RBAC and IAM

Module objectives

1. Create a namespace
1. Add a user to the cluster
1. Create roles and cluster roles and assign them to the user
1. Make sure that roles are enforced
1. Use IAM to control user permissions on a cluster level

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

```

