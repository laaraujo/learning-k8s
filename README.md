# learning-k8s

Repo I'm using to learn and experiment with the basics of Kubernetes in a local development environment.
For the purpose of simplicity this repo is using [mongo](https://hub.docker.com/_/mongo) and [mongo-express](https://hub.docker.com/_/mongo-express) docker images.


## Table of contents
- [Dependencies](#dependencies)
- [K8s](#kubernetes)
- [Pods](#pods)
- [Deployments](#deployments)
- [Services](#services)
- [Ingress](#ingress)
- [ConfigMap](#configmap)
- [Secrets](#secret)
- [Namespaces](#namespaces)
- [Persistent Volumes](#persistent-volumes)
- [Persistent Volume Claims)](#persistent-volume-claims)
- [Sources](#sources)

## Dependencies
* [Docker](https://www.docker.com/)
* [Minikube](https://minikube.sigs.k8s.io/)
* [Kubernetes and Kubectl](https://kubernetes.io/)

## Kubernetes

* Open source container orchestration tool
* Helps manage containerized applications in different environments

### How do we interact with a Kubernetes cluster ?

Master Nodes! Master nodes control the cluster state and the worker nodes.
There are 4 processes that run on every master node:

* [API Server](#api-server)
* [Scheduler](#scheduler)
* [Controller Manager](#controller-manager)
* [etcd](#etcd)

#### API Server
When we want to deploy a new application in a Kubernetes cluster we interact with the API server through some client (Kubernetes dashboard, CLI tool like Kubelet or the Kubernetes API).
It servers as a cluster gateway that gets the initial requests of any updates into the cluster and the queries to the cluster.

#### Scheduler
After the API server validates a request it handles it to the Scheduler.
It checks for current and desired resources and decides where to create/update/delete resources.

#### Controller Manager
Daemon process in charge of detecting state changes within the cluster (i.e. crashing `Pods`).
When a resource "dies" it will make a request to the Scheduler to re-schedule that resource.

#### etcd
Key-value store of the cluster state.
Any change in the cluster is stored here (except application data).


## Pod
This are the smallest deployable unit in K8s and an abstraction layer over Containers.<br>
**Generally we do NOT create these directly, instead they are created by resources like `Deployments` or `Jobs`.**

### Notes

* Ephemeral (they can "die" easily)
* Usually 1 application per Pod
* Each pod gets their IP address (internal)

### Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - name: mongodb
    image: mongo:6.0
    ports:
    - containerPort: 80
```

## Deployments
Declarative definition for `Pods` and `ReplicaSets`.
We can specify the desired:
* container image with the `.spec.template.spec.containers[n].image` attribute.
* number `Pods` with the `.spec.replicas` attribute
* environment variables with `.spec.template.spec.containers[n].env`

### Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
spec:
  replicas: 3 # default to 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo # https://hub.docker.com/_/mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef: 
              name: mongodb-secret # secret name
              key: mongo-root-username # secret data key
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret # secret name
              key: mongo-root-password # secret data key

```

## Services
Basically a static/permanent IP address that can be attached to pods in the corresponding network.
Its lifecycle is not connected to the referenced Pods.<br>
The resulting url usually looks like this: `protocol://node-ip-address:port`.

### Example
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb # spec.selector.matchLabels.app from Deployment template
  ports:
  - port: 27017
    targetPort: 27017
```

### Expose the service to the host machine with:
```sh
minikube service mongo-express-service
```

![running minikube service](./docs/minikube_service_terminal.png)
![service result](./docs/minikube_service_browser.png)

## Ingress
Manages external access to services in a cluster. It exposes HTTP/HTTPS routes from outside the cluster to services within it.

![ingress diagram](./docs/k8_ingress.png)

### Start a tunnel and use the correponding `host` in `Ingress` to access service:
```sh
minikube tunnel
```

![running minikube tunnel](./docs/minikube_tunnel_terminal.png)
![ingress result](./docs/minikube_tunnel_browser.png)

## ConfigMap
Maps external configuration of your application

### Example
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-configmap
data:
  database_url: mongodb-service
```

## Secret
Pretty much the same as ConfigMap but it is used to store secret data AND it is stored in base64 encoded format.

```yml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  mongo-root-username: dXNlcm5hbWU= # "username" base64 encoded
  mongo-root-password: cGFzc3dvcmQ= # "password" base64 encoded

```

## Namespaces

These provide a way to isolate groups of resources within a single cluster.
Resource names have to be unique within a namespace, but not across namespaces

### Default namespaces
Command: `kubectl get namespaces`

```logs
NAME              STATUS   AGE
default           Active   10h
kube-node-lease   Active   10h
kube-public       Active   10h
kube-system       Active   10h
```

#### `kube-system`
* Not meant to be interacted with
* System processes
* Master and Kubectl processes

#### `kube-public`
* Publicly accessidble data
* Has a config map that contains cluster information
    * i.e. `kubectl cluster-info`

#### `kube-node-lease`
* Info about hearbeats of nodes
* Lease object associated to each node in namespace
* Determines availability of each node

#### `default`
* Everything you create will default to this namespace

### Usage

#### Referencing resource from a different namespace

In the following ConfigMap `database_url` will point to `example-service` from `example-namespace`

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-configmap
data:
  database_url: example-service.example-namespace
```

#### Defining namespace for a resource

##### Through CLI command
```sh
kubectl apply -f example-deployment.yml --namespace=example-namespace
``` 

##### Through configuration file metadata
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-configmap
  namespace: example-namespace
data:
  database_url: example-service
```

#### Working with a specific namespace
`kubectl get all` will allways defer to the `default` namespace unless told otherwise with `-n namespace-name`.
There are tools like [`kubens`](https://github.com/ahmetb/kubectx) to have a more comfortable experience working with multiple namespaces

### Notes

* You usually can't share resources between namespaces. (i.e. each namespace will have to have their own `ConfigMap` to connect to a DB **even if** the config is the same)

* Some components are cluster-specific and can't be bound to a namespace (i.e. `Volumes` and `Nodes`)

## Persistent Volumes
Cluster resource that is used to store data. It assumes that you take care of your actual physical storage.

### Example
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-persistent-volume
spec:
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /mnt/mongodb
```

### Mounting a local folder with `minikube`
Command: `minikube mount <host_dir>:<cluster_dir>`

```logs
📁  Mounting host path ./mongo_volumes into VM as /mnt/mongodb ...
    ▪ Mount type:   9p
    ▪ User ID:      docker
    ▪ Group ID:     docker
    ▪ Version:      9p2000.L
    ▪ Message Size: 262144
    ▪ Options:      map[]
    ▪ Bind Address: 172.23.105.90:37415
🚀  Userspace file server: ufs starting
✅  Successfully mounted ./mongo_volumes to /mnt/mongodb

📌  NOTE: This process must stay alive for the mount to be accessible ...
```


## Persistent Volume Claims
These allow a Pod to "claim" a persistent volume (and takes care of "picking" the "correct" persistent volume).

### Example
```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-persistent-volume-claim
spec:
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
  volumeName: mongo-persistent-volume
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
spec:
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      securityContext:
        runAsUser: 999 # mongodb user
        fsGroup: 999 # mongodb user
      containers:
      - name: mongodb
        ...
        volumeMounts:
        - name: random-volume-mount-name
          mountPath: /data/db # as specified in mongo official docs and docker image info
        securityContext:
          privileged: true
      volumes:
      - name: random-volume-mount-name
        persistentVolumeClaim:
          claimName: mongodb-persistent-volume-claim
```

## Sources

* [Kubernetes official docs](https://kubernetes.io/docs/home/)
* [Kubernetes Tutorial for Beginners](https://www.youtube.com/watch?v=X48VuDVv0do) - by TechWorld with Nana
