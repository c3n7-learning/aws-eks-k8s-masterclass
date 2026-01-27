# Kubernetes Fundamentals - Imperative Commands Using `kubectl`

- Code: https://github.com/stacksimplify/kubernetes-fundamentals
- Slides: https://github.com/stacksimplify/kubernetes-fundamentals/tree/master/presentation
- Cheat Sheet: https://kubernetes.io/docs/reference/kubectl/quick-reference/

## 22. Kubernetes Architecture

Kubernetes is a portable, extensible, open-source platform for managing _containerized workloads_. It has these features out-of-the-box:

- Service discovery and load balancing
- Storage orchestration
- Automated rollouts and rollbacks
- Automatic bin packing
- Self healing
- Secret and configuraiton management.

Kubernetes has a master and worker nodes. All have a container-runtime.

#### K8s Master:

1. `container runtime` (docker)
2. `etcd`
   - stores the master and node information
   - Consistent and highly-available `key-value` store used as the k8s _backing store_ for all cluster data
   - It _stores_ all the masters and worker node information.

3. `kube-scheduler`
4. `kube-apiserver`
   - It acts as the _frontend_ for the k8s control plane. It _exposes_ the k8s API.
   - CLI tools (like `kubectl`), users and even master components (`scheduler`, `controller manager`, `etcd`) and worker node components like `kubelet` and others talk with this API server

5. `kube controller manager`
   - Controllers are _responsible for noticing_ and responding when nodes, containers or endpoints go _down_. They amke decisions to _bring up new containers_ in such cases.
   - **Node Controller**: Responsible for noticing and responding when _nodes go down_.
   - **Replication Controller**: Responsible for maintaining the _correct number of pods_ for every replication controller object in the system.
   - **Endpoints Controller**: Populates the endpoints object (that is, joins Services & Pods)
   - **Service Account & Token Controller**: Creates default accounts and API Access for _new namespaces_.

6. `cloud controller manager`
   - a k8s control plane component that embeds _cloud-specific control logic_.
   - It only runs controllers that are _specific_ to your cloud provider.
   - _On-Premise_ k8s clusters will not have this component.
   - _Node controller_: for _checking_ the cloud provider to dertermine if a node has been deleted in the cloud after it stops responding.
   - _Route controller_: for setting up _routes_ in the underlying cloud infrastructure.
   - _Service controller_: for creating, updating and deleting cloud provider _load balancer_.

#### K8s Worker Nodes:

1. `container runtime` (docker)
   - Is the _underlying software_ where we run all these k8s components
   - We are using _Docker_, but we have other runtime options like `rkt`, `continer-d`, etc.

2. `kubelet`
   - Is the _agent_ that runs on every node in the cluster
   - Is _responsible_ for making sure that containers are running in a Pod on a node

3. `kube-proxy`
   - Is a _network proxy_ that runs on each node in your cluster.
   - It maintains _network rules_ on nodes.
   - In short, these network rules _allow_ network communication to your pods from network sessions _inside or outside_ of your cluster

## 23. Kubernetes vs AWS EKS Architecture

The EKS Control Plane has:

- Container Runtime (Docker)
- `etcd`
- `kube-scheduler`
- `kube-apiserver`
- EKS Controller Manager
- Fargate Controller Manager

An EKS Managed Node Group has:

- Container Runtime (Docker)
- `kubelet`
- `kube-proxy`

EKS lets us focus only on Application Workloads. We don't need to worry about any of the components above.

## 24. Kubernetes Fundamentals - Introduction

**Definitions**:

1. Pods
   - Is a single instance of an application
   - Is the smallest object that you can create in k8s

2. ReplicaSet
   - Will maintain a stalbe set of replica Pods running at any given time
   - Is often used to guarantee the availability of a specified number of identical Pods.

3. Deployment
   - Runs multiple replcias of your application and automatically replaces any instances that fail or become unresponsive.
   - Performs Rollout & rollback changes to applications.
   - Deployments are well-suited for stateless applications

4. Service
   - Is an abstraction for pods, providing a stable, so called virtual IP (VIP) address
   - A service sits infront of a POD and acts as a load balancer.

**Imperative vs Declarative approach**:

- Imperative: Use `kubectl` to provision any of the above
- Declarative: Use `yaml` and `kubectl` to provision any of the above

## 25. Introduction to Kubernetes Pods

With k8s, our core goal will be to deploy our applications in the form of _containers_ on _worker nodes_ in a k8s cluster

- K8s _does not_ deploy containers directly on worker nodes.
- A container is _encapsulated_ in to a k8s object named **Pod**
- A Pod is a _single instance_ of an application
- A pod is the _smallest object_ that we can create in k8s

In addition:

- Pods generally have a _one to one_ relationship with containers.
- To scale up, we create new pods and to scale down, we delete the pod.

Multi-Container Pods

- We **cannot have** multiple containers of the same kind in a single POD.
- Example: Two NGINX containers in a single POD serving the same purpose _is not recommended_.
- Can we have multi-container pods? Yes, provided _they are not of the same kind_.
- Helper Containers (Side-car)
  - Data Pullers: Pull data required by the main container
  - Data Pushers: push data by collecting data from main container (logs)
  - Proxies: Writes static data to html files using helper container and reads using main controller.
- Communication:
  - The two containers can easily communicate with each other easily as they share the same _network space_
  - They can also easily share the _same storage space_
- Multi-Container pods are a _rare use-case_ and we won't explore that in this course.

## 27. Kubernetes Pods Demo

The instructor uses the `eks` nodes for this section, but i'll just use `k3d`

```shell
k3d cluster create hello-cluster --api-port 6550 -p "8081:80@loadbalancer" -p "8082:82@loadbalancer" --agents 2
```

Get worker nodes status:

```shell
kubectl get nodes

# Get worker node status with wide option
kubectl get nodes -o wide
```

To create a pod:

```shell
# Template
kubectl run <desired-pod-name> --image <container-image>

# Replace pod name, container image
kubectl run my-first-pod --image stacksimplify/kubenginx:1.0.0
```

When creating a pod:

- k8s created a pod.
- pulled the docker image from docker hub.
- Created the container in the pod.
- Started the container present in the pod.

List pods:

```shell
kubectl get pods

# Alias names for pods is po
kubectl get po

# list pods with wide option
kubectl get pods -o wide
```

Desribe Pod:

```shell
# To get list of pod names
kubectl get pods

# Describe the pod
kubectl describe pod <pod-name>
kubectl describe pod my-first-pod
```

Access Application:

- We can currently access this application only inside worker nodes.
- To access it externally, we need to create a _NodePort_ service.
- We will explore services later on.

Delete pod:

```shell
kubectl get pods

# Delet pod
kubectl delete pod <pod-name>
kubectl delete pod my-first-pod
```

## 28. Kubernetes NodePort Service

We can expose an application running on a set of pods using different types of services available in k8s

- ClusterIP
- NodePort
- LoadBalancer

NodePort Service

- To access our application _outside of k8s cluster_, we can use a NodePort servcie
- It exposes the service on each Worker Node's IP at a static port (nothing but NodePort)
- A ClusterIP Service, to which the NodePort service routes, is automatically created.
  - A ClusterIP service is automatically created when you create a NodePort service
  - Traffic is mapped like: `External Traffic → NodePort (32000) → ClusterIP Service (80) → Pod (80)`
- More precisely:
  - NodePort (32000): The port exposed on each node's external IP
  - Service Port (80): The port the ClusterIP service listens on within the cluster
  - TargetPort (80): The port on the actual pod/container
- Port range `30000`-`32767`

## 29. Kubernetes NodePort Service and Pods Demo

The instructor uses an EKS cluster, [but we'll use `k3d`](https://k3d.io/v5.3.0/usage/exposing_services/#2-via-nodeport):

- cluster name: `hello-cluster`
- number of nodes (agents): `2`
- publish a nodeport range.
- wait for 600s before giving up, as publishing an entier IP range could take a while.

```shell
k3d cluster create hello-cluster --agents 2 -p "30000-30050:30000-30050@server:0" --timeout=600s
```

Expose pod with a service (NodePort Service) to access the application externally (from the internet)

- Ports:
  - `port`: Port on which node port service listens in the k8s cluster internally
  - `targetPort`: The container port on which our application is running
  - `NodePort`: Worker node port on which we can access our application

```shell
# Create a pod
kubectl run <pod-name> --image <container-image>
kubectl run my-first-pod --image stacksimplify/kubenginx:1.0.0

# Expose Pod as a Servcie
kubectl expose pod <pod-name> --type=NodePort --port=80 --name=<service-name>
kubectl expose pod my-first-pod --type=NodePort --port=80 --name=my-first-service

# Get Service Info
kubectl get service
kubectl get svc

# Get public IP of Worker Nodes
kubectl get ndoes -o wide
```

k3d considerations:

- Since we are only exposing ports `30000` to `30050`, let's edit the nodeport to use only that.

```shell
kubectl edit svc my-first-service
```

- Then change the nodePort:

```yaml
# ... (other fields)
spec:
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30040 # Change this value
  selector:
    app: nginx
  type: NodePort
# ... (other fields)
```

Access the application using public IP. If using AWS EKS, allow all inbound traffic to the nodes via the `remote` security group:

```
http://<node1-public-ip>:<node-port>

# for k3d
http://localhost:30040
```

If a target port is not defined, by default and for convenience, the `targetPort` is set to the same value as the `port` field.

```shell
# This will fail when accessing the app, as service port (81) and container port (80) are different
kubectl expose pod my-first-pod --type=NodePort --port=81 --name=my-first-service-2

# Expose pod as a service with container port (--target-port)
kubectl expose pod my-first-pod --type=NodePort --port=81 --target-port=80 --name=my-first-service-3

# Get service info
kubectl get service
kubectl get svc

# Get public IP of Worker Nodes
kubectl get nodes -o wide

# remember k3d considerations (edit the services)
kubectl edit svc my-first-service-3
```

Access the application using public IP:

```
http://<node1-public-ip>:<node-port>
```

## 30. Interact with Pod - Connect to Container

Verify pod logs

```shell
# Get pod name
kubectl get po

# Dump pod logs
kubectl logs <pod-name>
kubectl logs my-first-pod

# Stream pods with -f option and access applation to see logs
kubectl logs -f <pod-name>
kubectl logs -f my-first-pod
```

Connect to a container in a pod

```shell
# connect to nginx container in a pod
kubectl exec -it <pod-name> -- /bin/hs
kubectl exec -it my-first-pod -- /bin/bash

# Execute some cmds in nginx container
ls
cd /usr/share/nginx/html
cat index.html
exit
```

Running individual commands in a container

```shell
kubectl exec -it <pod-name> -- <cmd>

# sample commands
kubectl exec -it my-first-pod -- env
kubectl exec -it my-first-pod -- ls
kubectl exec -it my-first-pod -- cat /usr/share/nginx/html/index.html
```

Get YAML output:

```shell
# get pod definition YAML output
kubectl get pod my-first-pod -o yaml

# get servcie definition YAML output
kubectl get service my-first-service -o yaml
```

Clean-Up

```shell
kubectl get pod my-first-pod -o yaml

# get service definition yaml output
kubectl get service my-first-service -o yaml
```

## 31. Delete Pod

Let us clean up

```shell
# get all objects in default namespace
kubectl get all

# delete services
kubectl delete svc my-first-service
kubectl delete svc my-first-service-2
kubectl delete svc my-first-service-3

# delete pod
kubectl delete pod my-first-pod

# get all objects in default namespace
kubectl get all
```

## 32. Kubernetes ReplicaSet - Introduction

These help us in achieving:

- High availability and reliability
- Scaling
- Load balancing
- Labels & Selectors

1. High Availability & Reliability:
   - A ReplicaSet's purpose is to maintain a _stable set of replica pods_ running at any given time.
   - If our _application crashes (any pod dies)_, it will _recreate_ the pod immediately to ensure the configured number of pods are running at any given time.

2. Load balancing:
   - To avoid overloading of traffic to a single pod, we can use _load balancing._
   - k8s provides pod load balancing _out of the box_ using Services for the pods which are part of a ReplicaSet
   - _Labels and Selector_ are the _key items_ which _ties_ al 3 together (Pod, ReplicaSet & Service), we will know in detail when we are writing YAML manifests for these objects.

3. Scaling
   - When load becomes too much for the number of existing pods, k8s enables us to easily _scale_ our application, adding additional pods as needed.
   - This is going to be seamless and super quick.

## 33. Kubernetes ReplicaSet - Review Manifests and Create ReplicateSet

The instructor uses an EKS cluster, [but we'll use `k3d`](https://k3d.io/v5.3.0/usage/exposing_services/#2-via-nodeport):

```shell
k3d cluster create hello-cluster --agents 2 -p "30000-30050:30000-30050@server:0" --timeout=600s
```

Create the `replicaset-demo.yaml` file to look like:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-helloworld-rs
  labels:
    app: my-helloworld
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-helloworld
  template:
    metadata:
      labels:
        app: my-helloworld
    spec:
      containers:
        - name: my-helloworld-app
          image: stacksimplify/kube-helloworld:1.0.0
```

Create a ReplicaSet file

```shell
kubectl create -f replicaset-demo.yaml
```

List ReplicaSets

```shell
kubectl get replicaset
kubectl get rs
```

Describe the newly created ReplicaSet

```shell
kubectl describe rs/<replicaset-name>

kubectl describe rs/my-helloworld-rs
# or
kubectl describe rs my-helloworld-rs
```

Get a list of running pods

```shell
kubectl get pods
kubectl describe pod <pod-name>

# get a list of pods with pod id and node in which it is running
kubectl get pods -o wide
```

Verify the owner of the pod, under the `name` tag under `ownerReferences`, we will find the ReplicaSet name to which this pod belongs to:

```shell
kubectl get pods <pod-name> -o yaml
kubectl get pods my-helloworld-rs-kbgf4 -o yaml
```

## 34. Kubernetes ReplicaSet - Expose and Test via Browser

Expose the ReplicaSet with a service (NodePort Service) to access the appilcation externally (from internet)

```shell
kubectl expose rs <ReplicaSet-Name> --type=NodePort --port=80 --target-port=8080 --name=<service-name-to-be-create>
kubectl expose rs my-helloworld-rs --type=NodePort --port=80 --target-port=8080 --name=my-helloworld-rs-service

# Get Service Info
kubectl get service
kubectl get svc

# remember k3d considerations (edit the services)
kubectl edit svc my-helloworld-rs-service

# Get public IP of worker Nodes
kubectl get nodes -o wide
```

Access the appication using Public IP

```
http://<node1-public-ip>:<node-port>/hello

# for k3d
http://localhost:30040/hello
```

Test ReplicaSet reliability or high availability.

- Whenever a Pod is accidentally terminated due to some application issue, ReplicaSet should auto-create that Pod to maintain desired number of Replicas configured to achieve high Availability.

```shell
# get pod name
kubectl get pods

# delete the pod
kubectl delete pod <pod-name>

# verify the new pod got created automatically
# (verify age and name of new pod)
kubectl get pods
```

Test ReplicaSet scalability feature. Update the `replicaset-demo.yaml` from 3 to 6 `replicas`:

```yaml
# before change
spec:
   replicas: 3

# after change
spec:
   replicas: 6
```

Update the ReplicaSet

```shell
# apply the latest changes to ReplicaSet
kubectl replace -f replicaset-demo.yaml

# verify if new pods got created
kubectl get pods -o wide
```

Delete ReplicaSet:

```shell
# template
kubectl delete rs <replicaset-name>
kubectl delete rs/<replicaset-name>

# delete the rs
kubectl delete rs my-helloworld-rs

# verify if ReplicaSet got deleted
kubectl get rs
```

Delete Service created for the ReplicaSet:

```shell
# delete service
kubectl delete svc <service-name>
kubectl delete svc/<service-name>

kubectl delete svc my-helloworld-rs-servcie

# verify if service got deleted
kubectl get svc
kubectl get all
```

## 35. Kubernetes Deployment - Introduction

Deployments

- We can create a **Deployment to rollout a ReplicaSet**.
- Whenever we want to change our application version, we simply **Update the Deployment**.
- In case of anything, we can **rollback the deployment**.
- We can **scale a deployment** by updating the nbr of replicas.
- We can also **pause and resume a deployment**.
- We can also get the **deployment status** when we update the deployment.
- **Clean-Up Policy**: up to 10 (by default) of previous deployments are kept in the history. We can go back to the previous versions in case of anything.
- **Canary Deployments**: if we want to add a new version of a deployment, traffic will be distributed to both old/new versions.

## 36. Kubernetes Deployment - Demo

Create deployment:

- Create deployment to rollult a ReplicaSet
- Verify Deployment, ReplicaSet & Pods

```shell
# create deployment
kubectl create deployment <deployment-name> --image=<container-image>
kubectl create deployment my-first-deployment --image=stacksimplify/kubenginx:1.0.0

# verify deployment
kubectl get deployments
kubectl get deploy

# describe deployment
kubectl describe deployment <deployment-name>
kubectl describe deployment my-first-deployment

# verify replicaset
kubectl get rs

# verify pod
kubectl get po
```

Scaling a deployment:

```shell
kubectl scale --replicas=20 deployment/<deployment-name>
kubectl scale --replicas=20 deployment/my-first-deployment

# verify deployment
kubectl get deploy

# verify replicaset
kubectl get rs

# verify pod
kubectl get po

# scale down the deployment
kubectl scale --replicas=5 deployment/my-first-deployment
kubectl get deploy
```

Expose deployment as a service

```shell
# expose the deployment
kubectl expose deployment <deployment-name> --type=NodePort --port=80 --target-port=80 --name=<service-name-to-create>
kubectl expose deployment my-first-deployment --type=NodePort --port=80 --target-port=80 --name=my-first-deployment-service

# get service info
kubectl get svc

# remember k3d considerations (edit the services)
kubectl edit svc my-first-deployment-service

# get public IP of worker nodes
kubectl get nodes -o wide
```

Access the application using public IP:

```
http://<node1-public-ip>:<node-port>/hello

# for k3d
http://localhost:30040/hello
```

## 37. Kubernetes Deployment - Update Deployment using Set Image Option

Update Deployment:

- Please check the container name in `spec.containers.name` yaml output and make a note of it and replica in `kubectl set image` command

```shell
# get container name from current deployment
kubectl get deployment my-first-deployment -o yaml

# update deployment
kubectl set image deployment<deployment-name> <container-name>=<container-image> --record=true
kubectl set image deployment/my-first-deployment kubenginx=stacksimplify/kubenginx:2.0.0 --record=true
```

Verify Rollout Status:

```shell
kubectl rollout status deployment/my-first-deployment

# verify deployment
kubectl get deploy
```

Describe deployment:

- Verify the Events and understand that k8s by default does "Rolling Updates" for new application releases.
- With that said, we will not have downtime for our application

```shell
kubectl describe deployment my-first-deployment
```

Verify ReplicaSet:

- New ReplicaSet will be created for new version

```shell
kubectl get rs
```

Verify Pods:

- pod tempalte hash label of new replicaset should be present for Pods letting us know that these pods belong to new ReplicaSet

```shell
kubectl get po
```

Verify Rollout history of a deployment:

```shell
kubectl rollout history deployment/my-first-deployment
```

Access the application using public IP:

```shell
# get nodeport
kubectl get svc

# get public IP of worker nodes
kubectl get nodes -o wide

# application url
# http://<worker-node-public-ip>:<node-port>
# http://localhost:30040
```

## 38. Kubernetes Deployment - Edit Deployment using kubectl edit

Update the application from V2 to V3 using "Edit Deployment" option

```shell
kubectl edit deployment<deployment-name> --record=true
kubectl edit deployment/my-first-deployment --record=true
```

Change the version from `2.0.0` to `3.0.0`:

```yaml
# change from 2.0.0
spec:
   containers:
      - image: stacksimplify/kubenginx:2.0.0

# change from 3.0.0
spec:
   containers:
      - image: stacksimplify/kubenginx:3.0.0
```

Verify Rollout Status:

```shell
kubectl rollout status deployment/my-first-deployment
```

Verify ReplicaSet & pods:

```shell
kubectl get rs
kubectl get po
```

Verify Rollout history of a deployment:

```shell
kubectl rollout history deployment/my-first-deployment
```

Access the application using public IP:

```
http://<node1-public-ip>:<node-port>

# for k3d
http://localhost:30040
```

## 39. Kubernetes Deployment - Rollback Application to Previous Version - Undo

Verify Rollout history of a deployment:

```shell
kubectl rollout history deployment/<deployment-name>
kubectl rollout history deployment/my-first-deployment
```

Verify changes in each revision. Review the "Annotations" and "Image" tags for clear understanding about changes

```shell
# list deployment history with revision information
kubectl rollout history deployment/my-first-deployment --revision=1
kubectl rollout history deployment/my-first-deployment --revision=2
kubectl rollout history deployment/my-first-deployment --revision=3
```

### Rollback to previous version:

If we rollback, it will go back to revision-2 and its number increases to revision-4

```shell
# undo deployment
kubectl rollout undo deployment/my-first-deployment

# list deployment
kubectl rollout history deployment/my-first-deployment
```

Verify deployment, pods, replicasets:

```shell
kubectl get deploy
kubectl get rs
kubectl get po
kubectl describe deploy my-first-deployment
```

Access the application using public IP:

```shell
# get nodeport
kubectl get svc

# get public IP of worker nodes
kubectl get nodes -o wide

# application url
# http://<worker-node-public-ip>:<node-port>
# http://localhost:30040
```

### Rollback to specific version:

Check the rollout history of a deployment

```shell
kubectl rollout history deployment/<deployment-name>
kubectl rollout history deployment/my-first-deployment
```

Rollback to specific version:

```shell
kubectl rollout undo deployment/my-first-deployment --to-revision=3
```

List deployment history

- If we rollback, it will go back to revision-3 and its number increases to revision-5

```shell
# list deployment
kubectl rollout history deployment/my-first-deployment
```

### Rolling restarts of application

Rolling restarts will kill the existing pods and recreate new pods in a rolling fashion

```shell
kubectl rollout restart deployment/<deployment-name>
kubectl rollout restart deployment/my-first-deployment

# get list of pods
kubectl get po
```

## 40. Kubernetes Deployment - Pause & Resume Deployments

This is used when we want to make multiple changes to our deployment. We can pause the deployment, make all changes, then resume it.

- We are going to update our application version from V3 to V4 as part of this section

### Pausing & Resuming Deployments

Check the current state of deployment & application:

```shell
# make note of last version number
kubectl rollout history deployment/my-first-deployment

# get list of replicasets
kubectl get rs

# access the application (make a note of app version)
# http://<worker-node-public-ip>:<node-port>
# http://localhost:30040
```

Pause deployment & make 2 changes:

```shell
# pause the deployment
kubectl rollout pause deployment/<deployment-name>
kubectl rollout pause deployment/my-first-deployment

# update deployment - application version from v3 to v4
kubectl set image deployment/my-first-deployment kubenginx=stacksimplify/kubenginx:4.0.0 --record=true

# check the rollout history of deployment (no new rollout)
kubectl rollout history deployment/my-first-deployment

# get list of replicasets (no new rs)
kubectl get rs

# make one more change
kubectl set resources deployment/my-first-deployment -c=kubenginx --limits=cpu=20m,memory=30Mi
```

Resume deployment:

```shell
kubectl rollout resume deployment/my-first-deployment

# check the rollout history of a deployment (new rollout)
kubectl rollout history deployment/my-first-deployment

# get list of replicasets (new rs)
kubectl get rs

# access the application (make a note of app version)
# http://<worker-node-public-ip>:<node-port>
# http://localhost:30040
```

Clean Up:

```shell
# delete deployment
kubectl delete deployment my-first-deployment

# delete servcie
kubectl delete svc my-first-deployment-service

# get all objects from k8s default namespace
kubectl get all
```

## 41. Kubernetes Services - Introduction

There are different types of services

- `ClusterIP` - Used for communication between applications inside a k8s cluster (e.g. frontend app accessing backend app)
- `NodePort` - Used for accessing applications outside of k8s cluster using worker node ports (e.g. accessing frontend application on browser)
- `LoadBalancer` - Primarily for cloud providers to integrate with their Load Balancer services (e.g. AWS Elastic Load Balancer)
- `Ingress` - Is an advanced load balancer which provides context path based routing, SSL, SSL Redirect and many more (e.g. AWS ALB)
- `externalName` - To access externally hosted apps in k8s cluster (e.g. Access AWS RDS database endpoint by applications present inside k8s cluster)

![k8s services](../media/04-1-k8s-services.png)

## 42. Kubernetes Services - Demo

We are going to look into ClusterIP and NodePort in this section with a default example

- `LoadBalancer` type is primarily for cloud providers and it will differ cloud to cloud, so we will do it accordingly.
- `ExternalName` does not have imperative commands and we need to write YAML definition for the same, so we will look into it as and when it is required in our course

### ClusterIP Service - Backend Application Setup

Create a deployment for Backend Application (Spring Boot REST Application).

- Create a ClusterIP service for load balancing backend application.
- If backend application port (Container Port: 8080) and Service Port (8080) are same we don't need to use `--target-port-8080` but for avoiding the confusion we add it.

```shell
# create deployment for backend rest app
kubectl create deployment my-backend-rest-app --image=stacksimplify/kube-helloworld:1.0.0
kubectl get deploy

# create a ClusterIP service for Backend Rest App
kubectl expose deployment my-backend-rest-app --port=8080 --target-port=8080 --name=my-backend-service
kubectl get svc

# we don't need to specify "--type=ClusterIP" because default setting is to create ClusterIP service
```

### NodePort Service - Frontend Application Setup

- We have implemented **NodePort Service** multiple times so far (in pods, replicasets and deployments). We are going to implement one more time to get a full architectural view in relation with ClusterIP service.
- Create a deployment for Frontend Application (nginx acting as reverse-proxy)
- Create a `NodePort` service for load balancing frontend application.
- **Important Note**: In Nginx reverse proxy, ensure backend service name `my-backend-service` is updated when you are building the frontend container. We already built it and put ready for this demo (`stacksimplify/kube-frontend-nginx:1.0.0`)
- The frontend service has this `nginx.conf` file:

```conf
server {
   listen       80;
   server_name  localhost;

   location / {
      # update your backend application k8s Cluster-IP service name & port below
      # proxy_pass http://<backend-clusterip-service-name>:<port>;
      proxy_pass http://my-backend-service:8080;
   }

   error_page 500 502 503 504 /50x.html;
   location = /50x.html {
      root /usr/share/nginx/html;
   }
}
```

- Docker image location: https://hub.docker.com/repository/docker/stacksimplify/kube-frontend-nginx
- Frontend nginx reverse proxy application source: https://github.com/stacksimplify/kubernetes-fundamentals/blob/master/00-Docker-Images/03-kube-frontend-nginx

```shell
# create deployment for frontend nginx proxy
kubectl create deployment my-frontend-nginx-app --image=stacksimplify/kube-frontend-nginx:1.0.0
kubectl get deploy

# create ClusterIP service for frontend nginx proxy
kubectl expose deployment my-frontend-nginx-app --type=NodePort --port=80 --target-port=80 --name=my-frontend-service
kubectl get svc

# Capture IP and Port to Access Application
kubectl get svc
kubectl get nodes -o wide
# http://<node1-public-ip>:<node-port>/hello

# remember k3d considerations (edit the services)
kubectl edit svc my-frontend-service

# scale backend with 3 replicas
kubectl scale --replicas=3 deployment/my-backend-rest-app

# test again to view the backend service load balancing
# http://<node1-public-ip>:<node-port>/hello
```
