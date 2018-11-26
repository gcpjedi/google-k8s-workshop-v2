Containers
==========

Module objectives
-----------------

1. Build application Docker image
1. Run application locally

---

The sample app
--------------

You'll use a very simple sample application—`gceme`—as the basis for your CD pipeline. `gceme` is written in Go and is located in the `sample-app` directory in this repo. When you run the `gceme` binary on a GCE instance, it displays the instance's metadata in a card:

![](docs/img/info_card.png)

The binary supports two modes of operation, designed to mimic a microservice. In backend mode, `gceme` will listen on a port (8080 by default) and return GCE instance metadata as JSON, with content-type=application/json. In frontend mode, `gceme` will query a backend `gceme` service and render that JSON in the UI you saw above. It looks roughly like this:

```
-----------      ------------      ~~~~~~~~~~~~        -----------
|         |      |          |      |          |        |         |
|  user   | ---> |   gceme  | ---> | lb/proxy | -----> |  gceme  | ----+
|(browser)|      |(frontend)|      |(optional)|   |    |(backend)|     |    +------+
|         |      |          |      |          |   |    |         |     |    |      |
-----------      ------------      ~~~~~~~~~~~~   |    -----------     +--->|  DB  |
                                                  |    -----------     |    |      |
                                                  |    |         |     |    +------+
                                                  |--> |  gceme  |-----+
                                                       |(backend)|
                                                       |         |
                                                       -----------
```

Both the frontend and backend modes of the application support two additional URLs:

1. `/version` prints the version of the binary (declared as a const in `main.go`)
1. `/healthz` reports the health of the application. In frontend mode, health will be OK if the backend is reachable.

A deployment is a supervisor for pods and replica sets, giving you fine-grained control over how and when a new pod version is rolled out as well as rolled back to a previous state.

Build application Docker image
------------------------------

1. Open the GCP console from your browser. [GCP Console](https://console.cloud.google.com/) and open the Cloud Shell.

1. Navigate to the `google-k8s-workshop/sample-app` folder

    ```
    $ cd google-k8s-workshop/sample-app
    ```

    `google-k8s-workshop` repository should be cloned in the previous exercise.

1. Set the `IMAGE` variable and build the Docker image

    ```shell
    export IMAGE=gcr.io/$PROJECT_ID/sample-k8s-app:1.0.0
    docker build . -t $IMAGE
    ```

    `gcr.io` is the repository hostname.

    `$PROJECT_ID` is the id of your GCP project

    `sample-k8s-app` is the name of the application image

    `1.0.0` is the image tag

    `docker build` command packages the application into a docker container. It does the following steps.
    * Reads the [Dockerfile](https://github.com/Altoros/google-k8s-workshop/blob/master/sample-app/Dockerfile#L15)
    * Creates a new container from the base image specified in the Dockerfile
    * Runs all commands from the Dockerfile
    * Saves the container filesystem as a new Docker image  

1. Push the image to the GCE container registry
    ```shell
    $ docker push $IMAGE
    ```
    No authentication is required because you are already authenticated by the Cloud Shell

1. In GCP console open 'Container Registry' -> 'Images' and make sure that `sample-k8s-app` image is present

Run application in the Cloud Shell
----------------------

1. Run database container

    ```
    $ docker run --rm \
      --name db \
      -e MYSQL_ROOT_PASSWORD=root \
      -d mysql
    ```

    `mysql` tells Docker to use `library/mysql:latest` image for the database from the `hub.docker.io` repository

    `-d` tells Docker to run the container in the background. If you need you can still use `docker logs` command to examine the container output

    `-e MYSQL_ROOT_PASSWORD=root` sets database administrator password to `root`

    `--name db` sets the name of the container which you can refer to from other commands

    `--rm` tells Docker to delete the container as soon as it is stopped or the root process inside container finishes execution

1. Run the backend container

    ```shell
    $ docker run --rm \
      --name backend \
      --link db:mysql \
      -p 8081:8081 \
      -d $IMAGE \
      app -port=8081 -db-host=db -db-password=root
    ```

    `--link db:mysql` [links](https://docs.docker.com/network/links/) the backend container to the database container

    `-p 8081:8081` expose port 8081 from the container as port `8081` on the host

    `$IMAGE` use image we build earlier for the sample app

    `app -port=8081 -db-host=db -db-password=root` application start command. `app` is the executable file that we build and package inside the container previously. In parameters, we specify that the app should listen at port `8081` and how it can connect to the database.

1. Run the frontend container

    ```shell
    $ docker run --rm \
      --name frontend \
      --link backend \
      -p 8080:8080 \
      -d $IMAGE \
      app  -frontend=true -backend-service=http://backend:8081
    ```

    Here we run the same executable but now we provide `-frontend=true` parameter which instructs the app to run in frontend mode. We also provide the connection string to that backend.

1. Verify that all containers are running

    ```
    $ docker ps
    CONTAINER ID        IMAGE                     COMMAND                  CREATED              STATUS              PORTS                    NAMES
    594fec987c57        lexsys27/sample-k8s-app   "app -frontend=true …"   8 seconds ago        Up 6 seconds        0.0.0.0:8080->8080/tcp   frontend
    684113a1910f        lexsys27/sample-k8s-app   "app -port=8081 -db-…"   21 seconds ago       Up 19 seconds       0.0.0.0:8081->8081/tcp   backend
    dd3bacf6e0f0        mysql                     "docker-entrypoint.s…"   About a minute ago   Up About a minute   3306/tcp, 33060/tcp      db
    ```
1. Click on the `Web preview` button in you Cloud Shell and then select `Preview on port: 8080` This will expose the app to your local machine from the Cloud Shell. See this [link](https://cloud.google.com/shell/docs/using-web-preview) for more details about web preview.

1. Check that the app is working

    The application will show some information about the VM that hosts the app. We take this information from [GCP instance metadata](https://cloud.google.com/compute/docs/storing-retrieving-metadata)

    You should be able to add notes in the bottom box. Notes were added to demonstrate how the app can handle persistent data (in our case we store them in the mysql database)

1. Clean up

    ```shell
    $ docker stop $(docker ps -aq)
    $ docker rm $(docker ps -aq)
    ```

Optional Exercises
-------------------

### Use external volume for mysql container

By default, mysql container stores its data inside the container file system. However, there is a possibility to store this data in a particular folder on the host and mount this folder to the container as a volume. Follow the instructions from the `Where to Store Data` section from the [mysql image documentation](https://hub.docker.com/_/mysql/) in order to do th at.
