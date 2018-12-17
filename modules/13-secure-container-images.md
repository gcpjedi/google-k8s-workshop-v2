# Securing Container Images

## Module Objectives

1. Enable GCR vulnerability scanning
1. View and filter vulnerability occurrences
1. Redeploy the cluster with Binary Authorization enabled
1. Configure and test custom Binary Authorization policy

---

## Enable GCR vulnerability scanning

Google Container Registry may scan images for known vulnerabilities. It will notify you by adding `Vulnerabilities` column in the image build list. You may extend the support further by enabling Cloud Pub/Sub queue notifications to trigger automated response and Binary Authorisation to prevent insecure images from running on top of Kubernetes cluster.

1. From Google Web Console go to Container Registry -> Settings.

1. In the Settings click `Enable Container Analysis API`.

    ![Enable API](img/gcr-enable.png)

    > Note: Container registry scanning will be enabled automatically.

1. Now each image screen will show the vulnerabilities per build.

    ![Vulnerabilities - list](img/security-images.png)

1. Click on the vulnerabilities column to show the list of vulnerabilities. Each has severity column which tells how dangerous is the bug. The package column shows the component of the image that has the bug and link to CVE describing it.

    > Note: As of writing, scans currently support Alpine, Ubuntu and Debian systems.

## View and filter vulnerability occurrences

1. To view summary of vulnerabilities for the application image you can use a `gcloud` command.

    ```shell
    gcloud beta container images list-tags --show-occurrences \
      gcr.io/project-altoros-workshop/sample-k8s-app \
      --occurrence-filter='kind="PACKAGE" AND has_prefix(resource_url, "gcr.io/project-altoros-workshop/sample-k8s-app")'
    ```

    ```
    DIGEST        TAGS   TIMESTAMP            VULNERABILITIES
    1dd9b5827cc3  1.0.2  2018-11-30T10:36:17  CRITICAL=5,HIGH=35,LOW=21,MEDIUM=184
    fbccfc8b25fe  1.0.0  2018-11-26T12:29:40  CRITICAL=5,HIGH=35,LOW=21,MEDIUM=184
    ```

    This example shows vulnerability summary for the image `sample-k8s-app` in the project `project-altoros-workshop`. Modify the request to show data on the image in your own repository.

    > Note: To get a vulnerabilities list, we need to talk to the API directly. There is no subcommands for the `gcloud` tool yet.

1. Get an authentication token. You will put it into each request so Google Cloud can authenticate it and grant the same rights your user has.

    ```shell
    gcloud config config-helper --format='value(credential.access_token)'
    ```

1. List vulnerability occurrences for your project.

    ```shell
    curl -s -XGET \
      -H "Authorization: Bearer $(gcloud config config-helper --format='value(credential.access_token)')" \
      https://containeranalysis.googleapis.com/v1beta1/projects/<PROJECT_ID>/occurrences?filter=kind%3D%22PACKAGE%22 | jq .
    ```

    * `curl` is a tool to make HTTP requests from the command line.
    * `-s` option tells to hide download statistics information.
    * `-XGET` sets the HTTP method `GET`, a method to read data.
    * `-H Authorization` sets the authorization header.
    * `https://containeranalysis.googleapis.com/v1beta1/` is API we are going to use.
    * `<PROJECT_ID>` is substituted forthe actual value of your project ID.
    * `filter=kind%3D%22PACKAGE%22` shows only occurrences of type `PACKAGE`.
    * `jq .` pretty-prints the json result.

1. Lets list the names of all the packages affected, modify and run the previous command.

    ```shell
    jq ".occurrences[].installation.installation.name"
    ```

    > Note: The package name is under `.installation.installation.name`, we can filter this with `jq`

    ```text
    "libss2"
    "procps"
    "python-configobj"
    "debconf"
    ```

There are less then 287 items in the list. Where are all the others? GCP API paginates the output so there is only a limited number of items in each page. If there are more items in the list the output has `nextPageToken` item. You may repeatedly query the results using the `pageToken` parameter until get all items. This exercise requires writing code so we omit it from the workshop.

## Redeploy the cluster with Binary Authorization enabled

1. Enable binary auth API.

    ```shell
    gcloud services enable binaryauthorization.googleapis.com
    ```

1. Create a cluster with binary API enabled

    ```shell
    gcloud beta container clusters create \
      --enable-binauthz \
      --zone us-west2-b \
      gke-workshop-1
    ```

1. Take a look at cluster default policy.

    ```shell
    gcloud beta container binauthz policy export  > manifests/policy.yaml
    cat manifests/policy.yaml
    ```

    Look through the policy [YAML reference doc](https://cloud.google.com/binary-authorization/docs/policy-yaml-reference).

1. Now check that you can deploy containers with Binary Auth enabled. The default security rule is `ALWAYS_ALLOW`.

    ```shell
    kubectl run nginx --image=nginx
    kubectl get pods
    ```

    ```
    NAME                   READY   STATUS    RESTARTS   AGE
    nginx-8586cf59-zjxl4   1/1     Running   0          4s
    ```

    ```
    kubectl delete deployment/nginx
    ```

### Deny Policy

Let's modify the policy to deny all the containers except needed by GKE cluster.

1. Modify the policy file `manifests/policy.yaml` and set `evaluationMode: ALWAYS_DENY`.

1. Update the policy.

    ```shell
    gcloud beta container binauthz policy import policy.yaml
    ```

1. Try to deploy again.

    ```shell
    kubectl run nginx --image=nginx
    kubectl get pods
    ```

    > Note: You will get a message `No resources found`, let’s get more information.

    ```
    kubectl describe rs -l run=nginx
    ```
    ```
    Warning  FailedCreate  41s  replicaset-controller  Error creating: pods "nginx-8586cf59-lbnjh" is forbidden: image policy webhook backend denied one or more images: Denied by default admission rule. Overridden by evaluation mode
    ```

    The policy works!

### Break Glass

There is a special case when you want to deploy without respect for the policy. It may happen during some emergencies.

1. Edit the `nginx` deployment.

    ```shell
    kubectl edit deployment/nginx
    ```

1. Add an annotation to the Pod spec (not Deployment).

    ```yaml
    annotations:
      alpha.image-policy.k8s.io/break-glass: "true"
    ```

1. The Pod will be created after you exit the editor.

    ```shell
    kubectl get pods --watch
    ```

    ```
    NAME                     READY   STATUS    RESTARTS   AGE
    nginx-7bf86b9677-sxqxm   1/1     Running   0          4s
    ```

    It is emergency scenario, typically for production related issues.

### Whitelist Docker library

What if I want to allow running all the images from the Docker official library?

1. Edit the `manifests/policy.yaml` and whitelist the Docker registry.

    All images from the library have the prefix `registry.hub.docker.com/library/*`.

1. Test the Deployment with `nginx` image.

    Because the controller doesn't know the `nginx` image comes from a standard registry, it just applies a regular expression filter!

1. Edit the Deployment and put the whole path to the image.

    ```yaml
    image: registry.hub.docker.com/library/nginx
    ```

    You should now see the container running.

1. Clean up by deleting the `nginx` Deployment .

    ```shell
    kubectl delete deployment/nginx
    ```

### Signing images

In this exercise you will set up an attestor. It is an entity-like CI/CD system that signs the images with PGP and clusters will accept only the images signed by all the attestors defined in the policy.

1. Export configuration variables.

    ```shell
    ATTESTOR=test-attestor
    NOTE_ID=test-attestor-note
    PROJECT_ID=project-altoros-workshop
    ```

    * `ATTESTOR` is the id of the attestor.
    * `NOTE_ID` is the id of a note (image tag) in the container registry

1. Create text file `note.json` with the json payload.

    ```json
    {
      "name": "projects/${PROJECT_ID}/notes/${NOTE_ID}",
      "attestation_authority": {
        "hint": {
          "human_readable_name": "Attestor Note"
        }
      }
    }
    ```

1. Create a note.

    ```shell
    curl -X POST \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer $(gcloud auth print-access-token)"  \
      --data-binary @note.json  \
      "https://containeranalysis.googleapis.com/v1beta1/projects/${PROJECT_ID}/notes/?noteId=${NOTE_ID}"
      {
        "name": "projects/${PROJECT_ID}/notes/test-attestor-note",
        "kind": "ATTESTATION",
        "createTime": "2018-12-03T14:08:45.821912Z",
        "updateTime": "2018-12-03T14:08:45.821912Z",
        "attestationAuthority": {
          "hint": {
            "humanReadableName": "Attestor Note"
          }
        }
      }
    ```

1. Verify the note was created.

    ```shell
    curl \
      -H "Authorization: Bearer $(gcloud auth print-access-token)" \
      "https://containeranalysis.googleapis.com/v1beta1/projects/${PROJECT_ID}/notes/${NOTE_ID}"
      {
        "name": "projects/${PROJECT_ID}/notes/test-attestor-note",
        "kind": "ATTESTATION",
        "createTime": "2018-12-03T14:08:45.821912Z",
        "updateTime": "2018-12-03T14:08:45.821912Z",
        "attestationAuthority": {
          "hint": {
            "humanReadableName": "Attestor Note"
          }
        }
      }
    ```

1. Create an attestor.

    ```shell
    gcloud beta container binauthz attestors create ${ATTESTOR} \
      --attestation-authority-note=${NOTE_ID} \
      --attestation-authority-note-project=${PROJECT_ID}
    ```

1. Verify the attestor was created.

    ```shell
    gcloud beta container binauthz attestors list
    ```

    > Note: Without a PGP keypair associated, the attestor can do nothing useful.

1. Generate a PGP keypair.

    ```shell
    gpg --batch --gen-key <(
      cat <<- EOF
        Key-Type: RSA
        Key-Length: 2048
        Name-Real: "Test Attestor"
        Name-Email: "test-attestor@example.com"
        %commit
    EOF
    )
    ```

1. List the keys.

    ```shell
    gpg --list-keys "test-attestor@example.com"
    ```

    ```
    gpg: checking the trustdb
    gpg: marginals needed: 3  completes needed: 1  trust model: pgp
    gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
    pub   rsa2048 2018-12-03 [SCEA]
          XXXXX
    uid           [ultimate] "Test Attestor" <"test-attestor@example.com">
    ```

1. Copy and save the public fingerprint (get from the output above).

    ```shell
    FINGERPRINT=XXXXX
    gpg --armor --export ${FINGERPRINT} > generated-key.pgp
    ```

1. Add the PGP key to the attestor.

    ```shell
    gcloud beta container binauthz attestors public-keys add \
        --attestor=${ATTESTOR} \
        --public-key-file=generated-key.pgp
    ```

1. Create a policy `manifests/policy-attestor.yml`.

    ```yaml
    name: projects/${PROJECT_ID}/policy
    admissionWhitelistPatterns:
    - namePattern: gcr.io/google_containers/*
    - namePattern: gcr.io/google-containers/*
    - namePattern: k8s.gcr.io/*
    - namePattern: gcr.io/stackdriver-agents/*
    defaultAdmissionRule:
      evaluationMode: REQUIRE_ATTESTATION
      enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
      requireAttestationsBy:
        - projects/${PROJECT_ID}/attestors/${ATTESTOR}
    ```

1. Import the policy.

    ```
    gcloud beta container binauthz policy import manifests/policy-attestor.yaml
    ```

1. Test the policy.

    ```shell
    kubectl apply -f manifests/frontend.yaml
    ```

    ```
    kubectl get pods
    ```

    > Note: You will get a message `No resources found`. This is okay as we still need to create attestation for the image to run it in the cluster.

1. Create an attestation.

    > Important: You will have different project-id and image signature!

    ```shell
    IMAGE_PATH="gcr.io/project-aleksey-zalesov/sample-k8s-app"
    IMAGE_DIGEST="sha256:1dd9b5827cc3da6db2b89d8d0a4bc5d4c1caa98a5efba62da61bfd206ce03af3"
    ```

    ```shell
    gcloud beta container binauthz create-signature-payload \
    --artifact-url=${IMAGE_PATH}@${IMAGE_DIGEST} > signature.json
    ```

    ```shell
    gpg \
      --local-user "test-attestor@example.com" \
      --armor \
      --sign signature.json
      --output signature.pgp \
    ```

    ```
    gcloud beta container binauthz attestations create \
      --artifact-url="${IMAGE_PATH}@${IMAGE_DIGEST}" \
      --attestor="projects/${PROJECT_ID}/attestors/${ATTESTOR}" \
      --signature-file=signature.pgp \
      --pgp-key-fingerprint="${FINGERPRINT}"
    ```

1. Verify the attestation.

    ```shell
    gcloud beta container binauthz attestations list \
        --attestor=$ATTESTOR --attestor-project=$PROJECT_ID
    ```

1. Start the sample application again.

    > Note: that this time you need to reference the image with the SHA256 sum instead of the tag, like this: `https://gcr.io/project-altoros-workshop/sample-k8s-app:1dd9b5827cc3da6db2b89d8d0a4bc5d4c1caa98a5efba62da61bfd206ce03af3`

    This time the application should run correctly.

1. Start the backend pod.

1. Create attestation for the MySQL image and run the `db` pod by yourself.

## Clean up

After doing these exercises, delete the cluster to free the resources.

```shell
gcloud beta container clusters delete \
  --zone us-west2-b \
  gke-workshop-1
```

---

Next: [Securing the Network and Container Runtime](14-secure-network.md)
