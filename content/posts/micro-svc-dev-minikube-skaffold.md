+++
title = "Micro-services development with Minikube and Skaffold"
date = 2020-11-01T21:16:24+08:00
description = "Tutorial on how to setup local development env for micro-services applications using minikube and skaffold"
categories = ['DevOps']
tags = ['Kubernetes', 'Minikube', 'Skaffold']
+++

While developing a multi-component application, like a micro-services backend, the release environments are quite different from the ones on our local machines. For example, during releases, the docker images are built and pushed to an artifactory then get deployed to a cloud kubernetes cluster. There are more steps going on in the CI/CD pipeline, accessing key vaults, applying configmaps, etc.

#### Scenario
Services are sometimes dependent on each other. As in my scenario, we are using the [API-LED architecture](https://blogs.mulesoft.com/dev/api-dev/what-is-api-led-connectivity/). Not like in the real kubernetes cluster, where backend components communicating as kubernetes service, I need to spin up the several dependent Node.js applications, assign them with different localhost ports and use `.env` config to make sure they connect.

In this case, we can use minikube to setup a local VM cluster and use Skaffold to automate build and deploy on it, to orchestrate services as if they work on a cloud cluster.

#### Create a local kubernetes cluster with minikube
First, we need to setup the minikube cluster ([downline guide](https://minikube.sigs.k8s.io/docs/start/)). Container or VM manager is required for minikube. 

After [Docker Desktop](https://www.docker.com/products/docker-desktop) is installed, on a mac with HomeBrew, simply run:
```bash
brew install minikube
```
Then install the `kubectl` cli tool if you haven't ([guide on other systems](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-macos)):
```bash
brew install kubectl 
```

To run the minikube container, first enable kubernetes on Docker and start the Docker daemon. Then run:
```bash
minikube start
```
The local cluster should be up and `kubectl` will be configured to use minikube by default (overriding your previous default config).

You can run `minikube dashboard` to launch the minikube dashboard service in the browser.

#### Create kubernetes deployments and services
Now that the cluster is up, it's time to run our server applications. Let's try manually deploying a service, we will bring in Skaffold later.

For each of my Node.js backend application, I will create a service deployment yaml config. For instance, for an authentication experience layer API server, a yaml file will be as followed:
```yaml
# auth-experience.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    app: auth-experience
  name: auth-experience
spec:
  replicas:
  selector:
    matchLabels:
      app: auth-experience
  template:
    metadata:
      labels:
        app: auth-experience
    spec:
      containers:
      - name: auth-experience
        image: demo/auth-experience # default tag 'latest'
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
        protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: auth-experience
  name: auth-experience
spec:
  type: ClusterIP
  selector:
    app: auth-experience
  ports:
  - name: auth-experience
    port: 80
    targetPort: 3000
    protocol: TCP
```
By applying this config, a service using the latest image tag of `demo/auth-experience` with be created. Note that we need to set `imagePullPolicy` to `Never` to tell kubectl to use local image only. 

Now we need to build the images for minikube cluster. Before running docker build, we need to run: 
```bash
eval $(minikube docker-env)
```

The purpose of this command, is to point your shell to minikube's docker daemon.

This `eval` command is explained here [Stack Overflow](https://stackoverflow.com/questions/52310599/what-does-minikube-docker-env-mean#:~:text=The%20command%20minikube%20docker%2Denv,and%20put%20them%20into%20effect.):
> The command  `minikube docker-env`  returns a set of Bash environment variable exports to configure your local environment to re-use the Docker daemon inside the Minikube instance.
> Passing this output through  `eval`  causes bash to evaluate these exports and put them into effect. 

If your run `minikube -p minikube docker-env`, you can inspect what env variables will be exported by the `eval` action.
```bash
$ minikube -p minikube docker-env
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://127.0.0.1:32770"
export DOCKER_CERT_PATH="/Users/roytang/.minikube/certs"
export MINIKUBE_ACTIVE_DOCKERD="minikube"
```

Run
```bash
docker images
```
to list images inside minikube.

Now you can build the image by running:
 
```bash
docker build -t <your-image-name-tag> /path/to/your/dockerfile
```

List images again and you should see the image's there.

#### Expose the kubernetes services
To apply the deployment, run:
```bash
kubectl apply -f /path/to/auth-experience.yaml
```

This service is now being created with the type of `ClusterIP`, which means they are accessible only within the cluster by default, by accessing the name of the service as host, i.e `http://auth-experience:80`.

There are a few ways to expose services from minikube to external access, for example, to you machine's localhost. They are mentioned in the [documentation](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types).

Or we can simply run:
```bash
minikube service auth-experience #<name-of-service>
```

A random localhost port will be assigned to tunnel the traffic into the service inside the minikube cluster.

#### Development workflow with Skaffold
For local development, we can further use Skaffold to watch our code and do the hot-reloading for us, so that we won't repeat the manual build-and-deploy process in the above section.

##### Skaffold [Installing](https://skaffold.dev/docs/install/)
```bash
brew install skaffold
```

##### Skaffold [init](https://skaffold.dev/docs/workflows/getting-started-with-your-project/) and [dev](https://skaffold.dev/docs/workflows/dev/)
The `skaffold init` command will look for k8s manifest for deployment configs and look for Dockerfile for building resources specified in your manifest files.

For every image you specify in the manifest, the init command will prompt you for selecting the corresponding Dockerfile under you current directory. Then the `skaffold.yaml` config file is generated for you.

A sample project structure is as followed:
```bash
# working directory tree
.
├── apps
│   ├── auth-experience
│   │   └── Dockerfile
│   ├── service-1
│   │   └── Dockerfile
│   └── service-2
│       └── Dockerfile
├── k8s-manifests
│   ├── auth-experience.yaml
│   ├── service-2.yaml
│   └── service-3.yaml
└── skaffold.yaml
```

The `skaffold.yaml` generated for this sample project will look like:
```yaml
apiVersion: skaffold/v2beta7
kind: Config
metadata:
  name: src
build:
  artifacts:
  - image: demo/auth-experience
    context: apps/auth-experience
  - image: demo/service-1
    context: apps/service-1
  - image: demo/service-2
    context: apps/service-2
deploy:
  kubectl:
    manifests:
    - k8s-manifest/auth-experience.yaml
    - k8s-manifest/service-1.yaml
    - k8s-manifest/service-2.yaml
```

To start coding your micro-service component apps, run:
```bash
skaffold dev
```

Skaffold will now watch you apps' source code, re-build and re-deploy your apps to the minikube cluster.

This should be the minimal config for setting up the local development env for micro-services on a kubernetes cluster. We may use `Kustomize` for more sophisticated configs and managing namespaces and releases workflow.

#### Other references
* [Kustomize](https://kustomize.io/)
* [tree cli tool for mac](https://rschu.me/list-a-directory-with-tree-command-on-mac-os-x-3b2d4c4a4827)