# Securing Container Images

## Module Objectives

1. Enable GCR Vulnerability Scanning
1. View and filter vulnerability occurrences
1. Redeploy the cluster with Binary Authorization enabled
1. Configure and test custom binary authorization policy

---

## Enable GCR Vulnerability Scanning

Google container registry may scan images for known vulnerabilities. It will notify you by adding `Vulnerabilities` columen in the image build list. You may extend the support further by enabling Cloud Pub/Sub queue notifications to trigger automated response and Binary Authorisation to prevent insecure images from running on top of Kubernetes cluster.

1. From Google Web Console go to Container Registry -> Settings

1. In the Settings click `Enable Container Analysis API`

    ![Enable API](img/gcr-enable.png)

    Container registry scanning will be enabled automatically.

1. Now each image screen will show the vulnerabilities per build.

    ![Vulnerabilities - list](img/security-images.png)

1. Click on the vulnerabilities column to show the list of vulnerabilities. Each has severity column which tells how dangerous is the bug, package column - the component of the image that has the bug and link to CVE describing it.

Scans currently work for Alpine, Ubuntu and Debian systems.

## View and filter vulnerability occurrences

To view summary of vulnerabilities for the application image one may use `gcloud` command.

```shell
$ gcloud beta container images list-tags --show-occurrences \
  gcr.io/project-aleksey-zalesov/sample-k8s-app \
  --occurrence-filter='kind="PACKAGE" AND has_prefix(resource_url, "gcr.io/project-aleksey-zalesov/sample-k8s-app")'
DIGEST        TAGS   TIMESTAMP            VULNERABILITIES
1dd9b5827cc3  1.0.2  2018-11-30T10:36:17  CRITICAL=5,HIGH=35,LOW=21,MEDIUM=184
fbccfc8b25fe  1.0.0  2018-11-26T12:29:40  CRITICAL=5,HIGH=35,LOW=21,MEDIUM=184
```

This example shows vulnerability summary for the image `sample-k8s-app` in the project `project-aleksey-zalesov`. Modify the request to show data on the image in your own repository.

Unfortunately, to get vulnerabilities list you will need to talk to the API directly. There is no subcommands for `gcloud` tool yet.

First grab authentication token. You will put it into each request so Google Cloud can authenticate it and grant the same rights your user has.

```shell
gcloud config config-helper --format='value(credential.access_token)'
```

```shell
curl -s \
  -XGET \
  -H"Authorization: Bearer $(gcloud config config-helper --format='value(credential.access_token)')" \ https://containeranalysis.googleapis.com/v1beta1/projects/<PROJECT_ID>/occurrences?filter=kind%3D%22PACKAGE%22 | jq .
```

`curl` is a toool to make HTTP requests from the command line.

`-s` option tells to hide download statistics information

`-XGET` sets the HTTP method `GET`. You typically use `GET` method to retrieve objects and `POST` to create/update the object. `DELETE` method is used to delete the object.

Next line sets the authentication header.

`https://containeranalysis.googleapis.com/v1beta1/` is API you going to use

Substitute `<PROJECT_ID>` to the actual value of your project ID.

`filter=kind%3D%22PACKAGE%22` show only occurences of type `PACKAGE`.

`jq .` pretty-prints the json result

What if I want to list the names of all the packages affected?

The package name is under `.installation.installation.name`. To filter it with `jq` use `jq ".occurrences[].installation.installation.name"`

```text
"libss2"
"procps"
"python-configobj"
"debconf"
..
```

Well, there is much less then 287 items in the list. Where are all the others? GCP API paginates the output so there is only a limited number of items in each page. If there are more items in the list the output has `nextPageToken` item. One may repetedly query the results using `pageToken` parameter until get all items. This exercise requires writing code so we omit it from the workshop.

## Redeploy the cluster with Binary Authorization enabled

1. Enable binary auth API

    ```shell
    gcloud services enable binaryauthorization.googleapis.com
    ```

1. Create a cluster with binary API enabled

    ```shell
    gcloud beta container clusters create \
        --enable-binauthz \
        --zone europe-west1-d \
        gke-workshop-1
    ```

1. Take a look at cluster default policy

    ```shell
    gcloud beta container binauthz policy export  > policy.yaml
    cat policy.yaml
    ```

    Look through the policy [YAML reference doc](https://cloud.google.com/binary-authorization/docs/policy-yaml-reference).

1. Now check that you can deploy containers with Binary Auth enabled. This due to default security rule is `ALWAYS_ALLOW`

    ```shell
    # test you can deploy with default policy
    $ kubectl run nginx --image=nginx

    $ kubectl get pods
    NAME                   READY   STATUS    RESTARTS   AGE
    nginx-8586cf59-zjxl4   1/1     Running   0          4s

    $ kubectl delete deployment/nginx
    deployment.extensions "nginx" deleted
    ```

### Deny policy

Let's modify the policy to deny all the containers except needed by GKE cluster.

1. Modify the policy file: set `evaluationMode: ALWAYS_DENY`

1. Update the policy

    ```shell
    gcloud beta container binauthz policy import policy.yaml
    ```

1. Try to deploy again

    ```shell
    # test you can deploy with updated policy
    $ kubectl run nginx --image=nginx

    $ kubectl get pods
    No resources found.

    # why?
    $ kubectl describe rs -l run=nginx
    ..
    Warning  FailedCreate  41s               replicaset-controller  Error creating: pods "nginx-8586cf59-lbnjh" is forbidden: image policy webhook backend denied one or more images: Denied by default admission rule. Overridden by evaluation mode
    ```

The policy works!

### Break glass

There is a special case when you want to deploy without respect for the policy. It may happen during some emergencies.

1. Edit `nginx` deployment

    ```shell
    kubectl edit deployment/nginx
    ```

1. Add annotation to the Pod spec (not Deployment one!)

    ```yaml
    annotations:
      alpha.image-policy.k8s.io/break-glass: "true"
    ```

1. Pod will be created after you exit the editor

    ```shell
    $ kubectl get pods --watch
    NAME                     READY   STATUS    RESTARTS   AGE
    nginx-7bf86b9677-sxqxm   1/1     Running   0          4s
    ```

It is emergency scenario. What if I want to allow running all the images from the Docker official library?

### Whitelist Docker library

Edit the policy and whitelist the Docker library. All images from the library have the prefix `registry.hub.docker.com/library/*`.  Don't forget to update the policy in the cloud. Then test the deployment with `nginx` image. What happened?

Because the controller doesn't know the nginx image comes from a standard library - it just applies regular expression filter! Let's edit the deployment and put the whole path to the image

```yaml
image: registry.hub.docker.com/library/nginx
```

Now you should see the container running.

Clean up

```shell
$ kubectl delete deployment/nginx
deployment.extensions "nginx" deleted
```

### Signing the images

In this exercise you will set up an attestor. Entity like CI/CD system that signs the images with PGP and cluster will accept only the images signed by all the attestors defined in the policy.

1. Export configuration variables

```shell
ATTESTOR=test-attestor
NOTE_ID=test-attestor-note
PROJECT_ID=project-aleksey-zalesov
```

`ATTESTOR` is the id of the attestor

`NOTE_ID` is the id of a note (image tage) in the container registry

1. Create text file with the note payload

    ```shell
    cat > note_payload.json << EOM
    {
      "name": "projects/${PROJECT_ID}/notes/${NOTE_ID}",
      "attestation_authority": {
        "hint": {
          "human_readable_name": "Attestor Note"
        }
      }
    }
    EOM
    ```

1. Create a note

    ```shell
    $ curl -X POST \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $(gcloud auth print-access-token)"  \
        --data-binary @./note_payload.json  \
        "https://containeranalysis.googleapis.com/v1beta1/projects/${PROJECT_ID}/notes/?noteId=${NOTE_ID}"
    {
      "name": "projects/project-aleksey-zalesov/notes/test-attestor-note",
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

1. Verify the note was created

    ```shell
    $ curl \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://containeranalysis.googleapis.com/v1beta1/projects/${PROJECT_ID}/notes/${NOTE_ID}"
    {
      "name": "projects/project-aleksey-zalesov/notes/test-attestor-note",
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

1. Create an attestor

    ```shell
    gcloud beta container binauthz attestors create ${ATTESTOR} \
    --attestation-authority-note=${NOTE_ID} \
    --attestation-authority-note-project=${PROJECT_ID}
    ```

    Verify the attestor was created

    ```shell
    gcloud beta container binauthz attestors list
    ```

    Without PGP keypair associated attestor can do nothing useful.

1. Generate PGP kaypair

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

1. List the keys

    ```shell
    $ gpg --list-keys "test-attestor@example.com"

    gpg: checking the trustdb
    gpg: marginals needed: 3  completes needed: 1  trust model: pgp
    gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
    pub   rsa2048 2018-12-03 [SCEA]
          XXXXX
    uid           [ultimate] "Test Attestor" <"test-attestor@example.com">
    ```

1. Copy and save the public fingerprint (get from the output above)

    ```shell
    FINGERPRINT=XXXXX
    gpg --armor --export ${FINGERPRINT} > generated-key.pgp
    ```

1. Add PGP key to the attestor

    ```shell
    gcloud beta container binauthz attestors public-keys add \
        --attestor=${ATTESTOR} \
        --public-key-file=generated-key.pgp
    ```

1. Create a policy

    ```shell
    cat > ./policy.yaml << EOM
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
        name: projects/${PROJECT_ID}/policy
    EOM

    gcloud beta container binauthz policy import policy.yaml
    ```

1. Test the policy

    ```shell
    $ kubectl apply -f manifests/frontend.yaml

    $ kubectl get pods
    No resources found.
    ```

    One need to create attestation for the image to run it in the cluster.

1. Create an attestation

    ```shell
    # you will have different project-id and image signature
    IMAGE_PATH="gcr.io/project-aleksey-zalesov/sample-k8s-app"
    IMAGE_DIGEST="sha256:1dd9b5827cc3da6db2b89d8d0a4bc5d4c1caa98a5efba62da61bfd206ce03af3"

    gcloud beta container binauthz create-signature-payload \
    --artifact-url=${IMAGE_PATH}@${IMAGE_DIGEST} > generated_payload.json

    # sign the payload
    gpg \
        --local-user "test-attestor@example.com" \
        --armor \
        --output generated_signature.pgp \
        --sign generated_payload.json

    # create the attestation
    gcloud beta container binauthz attestations create \
        --artifact-url="${IMAGE_PATH}@${IMAGE_DIGEST}" \
        --attestor="projects/${PROJECT_ID}/attestors/${ATTESTOR}" \
        --signature-file=generated_signature.pgp \
        --pgp-key-fingerprint="${FINGERPRINT}"

    # verify
    gcloud beta container binauthz attestations list \
        --attestor=$ATTESTOR --attestor-project=$PROJECT_ID
    ```

1. Start a sample application again.

    Note that this time you need to reference the image with SHA256 sum instead of tag, like this: `https://gcr.io/project-aleksey-zalesov/sample-k8s-app:1dd9b5827cc3da6db2b89d8d0a4bc5d4c1caa98a5efba62da61bfd206ce03af3`

    This time the application should run correctly.

    Start the backend pod, create attestation for the MySQL image and run db pod by yourslef.

## Clean up

After doing this execises delete the cluster to free the resources.

```shell
gcloud beta container clusters delete \
    --zone europe-west1-d \
    gke-workshop-1
```
