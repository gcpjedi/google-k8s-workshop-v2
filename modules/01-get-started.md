# Getting Started with GCP

## Module Objectives

Before getting started, you first have to prepare the environment for the workshop.

1. Get a GCP account from the instructor
1. Connect to the Cloud Shell using the GCP account
1. Enable the necessary APIs
1. Set computing zone and project
1. Download the lab source code from GitHub

---

## Google Cloud Platform Overview

- Managed by Google
- Provides basic resources like compute, storage and network
- Also provides services like Cloud SQL and Kubernetes engine
- All operations can be done through the API
- SLAs define reliability guarantees for the APIs
- Three ways of access
  - API calls
  - SDK commands
  - Cloud Console web UI

Google Cloud Computing service groups:

- Compute
- Storage
- Migration
- Networking
- Databases
- Developer Tools
- Management Tools

You will use these services while doing the lab:

- Kubernetes Engine: Create Kubernetes cluster
- IAM & Admin: Manage users and permissions
- Compute Engine: Run virtual machines for worker nodes
- VPC Network: Connectivity between the nodes
- Load Balancing: Create Ingress of LoadBalancer type
- Persistent Disk: Persistent volume for Jenkins
- Source Repositories: Hosting source code for an app
- Cloud Build: Build Docker containers
- Container Registry: Storing versioned Docker images of an app

Cloud Console is the admin user interface for Google Cloud. With Cloud Console you can find and manage your resources through a secure administrative interface.

Cloud Console features:

- Resource Management
- Billing
- SSH in Browser
- Activity Stream
- Cloud Shell

Cloud SDK provides essential tools for cloud platform.

- Manage Virtual Machine instances, networks, firewalls, and disk storage
- Spin up a Kubernetes Cluster with a single command

Projects

- Managing APIs
- Enabling billing
- Adding and removing collaborators
- Managing permissions for GCP resources

Zonal, Regional, and Global Resources

- Zone: Instances and persistent disks
- Region: Subnets and addresses
- Global: VPC Network and firewall

---

## Google Cloud Platform (GCP) Account

In this workshop you will run Kubernetes in GCP. We have created a separate project for each student. You should receive an email with the credentials to log in.

We recommend using Google's Chrome browser during the workshop.

1. Go to https://console.cloud.google.com/
1. Enter the username
1. Enter the user password

    > Note: Sometimes GCP asks for a verification code when it detects logins from unusual locations. It is a security measure to keep the account protected. If this happens, please ask the instructor for the verification code.

1. In the top left corner select the project "XXXXXXXXXXXXX-yyyyyy", where XXXXXXXXXXXXX matches the name of the e-mail you were given

## Cloud Shell

Console is the UI tool for managing cloud resources. Most of the exercises in this course are done from the command line, so you will need a terminal and an editor.

Click "Activate Cloud Shell" button in the top right corner.

![](img/cloud-shell.png)

Now click the "Start Cloud Shell" button in the lower right of the dialog.

This will start a virtual machine in the cloud and give you access to a terminal and an editor.

## Set Computing Zone and Region

When the shell is open, set your default compute zone and region:

```shell
export PROJECT_ID=$(gcloud config get-value project)

export COMPUTE_REGION=us-west2
gcloud config set compute/region $COMPUTE_REGION

export COMPUTE_ZONE=us-west2-b
gcloud config set compute/zone $COMPUTE_ZONE
```

Every time you open a new terminal you will need to input these commands. To avoid this, place the above commands inside `~/.profile` file and they will be executed automatically each time you log in.

> Note: changing the zone will not change the region automatically.

You can check for additional information with:
```shell
gcloud info
```

## Enable APIs

As a project owner, you control which APIs are accessible for the project. Enable the APIs which are required for the workshop:

```shell
gcloud services enable --async \
  container.googleapis.com \
  compute.googleapis.com \
  containerregistry.googleapis.com \
  cloudbuild.googleapis.com \
  sourcerepo.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  stackdriver.googleapis.com
```

The operation runs asynchronously. You can check if the APIs are enabled for the project, but enabling all these apis will take about 5m.

```shell
gcloud services list --enabled
```

You can also connect to status of job by running command suggested:

```shell
gcloud beta services operations wait operations/acf.xxxx-xxxx-xxxx-xxxx-xxxx
```

Once that completes or you waited about 5 minutes you can check services again:

```shell
gcloud services list --enabled
```

```
NAME                              TITLE
bigquery-json.googleapis.com      BigQuery API
cloudbuild.googleapis.com         Cloud Build API
compute.googleapis.com            Compute Engine API
container.googleapis.com          Kubernetes Engine API
containerregistry.googleapis.com  Container Registry API
logging.googleapis.com            Stackdriver Logging API
monitoring.googleapis.com         Stackdriver Monitoring API
oslogin.googleapis.com            Cloud OS Login API
pubsub.googleapis.com             Cloud Pub/Sub API
sourcerepo.googleapis.com         Cloud Source Repositories API
stackdriver.googleapis.com        Stackdriver API
storage-api.googleapis.com        Google Cloud Storage JSON API
```

Validate count:

```shell
gcloud services list --enabled|grep -v NAME|wc -l
```

Retry using sync mode if some APIs are not enabled.

```shell
gcloud services enable compute.googleapis.com <...>
```

```
Waiting for async operation operations/tmo-acf.8c2c26e0-4997-4378-964f-fdce6d0b9fec to complete...
Operation finished successfully. The following command can describe the Operation details:
 gcloud services operations describe operations/tmo-acf.8c2c26e0-4997-4378-964f-fdce6d0b9fec
```

```shell
gcloud services list --enabled | grep compute
```

## Download the Lab Source Code from GitHub

Clone the lab repository in your cloud shell, then `cd` into that directory:

```shell
git clone https://github.com/Altoros/google-k8s-workshop-v2.git
```
```shell
cd google-k8s-workshop-v2
```

---

Next: [Containers](02-containers.md)
