# Kubernetes Secrets, Init Containers, LivenessProbes, Request Limits & Namespaces

## 65. Kubernetes Important Concepts for Application Deployments -Introduction

We are going to explore:

- Secrets: to encrypt things like `DB_PASSWORD`.
- `Init Containers`: to ensure that mysql container comes up first before we start our user management microservice
- `Liveness & Readiness Probes`: only allow traffic to a pod if these probs yield success
- `Requests` and `Limits`: hardware resources and request limits
- `Namespaces`: to isolate things like frontend/backend, `dev`/`uat`/`production`

Reference:

- https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/05-Kubernetes-Important-Concepts-for-Application-Deployments

## 66. Kubernetes Secrets

- These let you store and manage sensitive information, such as password, OAuth tokens, and ssh keys.
- Storing confidential information is a secret is safer and more reliable than putting it directly in a Pod definition or in a container image.

Create secret for MySQL DB Passwor

```shell
# on *nix systems
echo -n 'dbpassword11' | base64

# URL: https://www.base64encode.org
```

Create k8s secrets manifest

- `type: Opaque` means that from k8s' POV, the contents of this Secret are unstructured. It can contain arbitrary key-value pairs

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-db-password
type: Opaque
data:
  # Output from echo -n 'dbpassword11' | base65
  db-password: ZGJwYXNzd29yZDEx
```

Update secret in MySQL Deployment for DB Password

```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-db-password
        key: db-password
```

Update secret in UMS Deployment (user magt service):

```yaml
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-db-password
      key: db-password
```

Create & Test:

```shell
# create all objects
kubectl apply -f 66-secrets/

# list pods
kubectl get pods

# access application health status page
http://<WorkerNode-Public-IP>:30030/usermgmt/health-status
```

Clean-Up:

```shell
# delete all
kubectl delete -f 66-secrets/

# list pods
kubectl get pods

# verify sc, pvc, pv
```

## 67. Kubernetes Init Containers

Introduction

- These run **before** App containers.
- The can contain **utilities** or **setup scripts** not present in an app image
- We can have and run **multiple init containers** before an App Container
- They are exactly like regular containers, except:
  - Init containers always _run to completion_
  - Each init container must _complete successfully_ before the next one starts
- If a Pod's init container fails, k8s repeatedly restarts the Pod until the init container succeeds
- However, if the Pod has a `restartPolicy: Never`, k8s does not restart the pod

Reference:

- https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

### Implement Init Containers

Update `initContainers` section under Pod Template Spec which is `spec.template.spec` in a Deployment

```yaml
template:
  metadata:
    labels:
      app: usermgmt-restapp
    spec:
      initContainers:
        - name: init-db
          image: busybox:1.31
          command:
            [
              "sh",
              "-c",
              'echo -e "Checking for the availability of the MySQL server"; while ! nc -z mysql 3306; do sleep 1; printf "-"; done; echo -e "  >> MySQL DB Server has started";',
            ]
```

Create and test:

```shell
# Create All Objects
kubectl apply -f 67-init-containers/

# list pods
kubectl get pods

# watch list pods screen
kubectl get pods -w

# describe pod & discuss about init container
kubectl describe pod <usermgmt-microservice-xxxxx>

# Access application health status page
http://<WorkerNode-Public-IP>:31231/usermgmt/health-status
```

Clean-Up
Delete all k8s objects created as part of this section

```shell
kubectl delete -f 67-init-containers/

# list pods
kubectl get pods

# verify sc, pvc, pv
kubectl get sc,pvc,pv
```

## 68. Kubernetes Liveness & Readiness Probes Introduction

We have 3 types of probes:

1. Liveness Probe
   - Kubelet uses liveness probes to know _when to restart a container_.
   - Liveness probes could catch a _deadlock_, where an application is running, but unable to make progress and _restarts the container_ to help in such cases.
2. Readiness Probe
   - Used by kubelet to know when _a container is ready to accept traffic_.
   - When a Pod is not ready, it is _removed_ from service load balancers based on this _readiness probe signal_.
3. Startup Probe
   - Kubelet uses startup probes to know when _a container application has started_.
   - This probe _disables_ liveness and readiness checks until it _succeeds_ ensuring those pods don't interfere with app startup.
   - This can be used to adopt liveness checks on _slow starting containers_, avoiding them getting killed by the kubelet before they are up and running.

Options to define Probes:

1. Check using Commands: `/bin/sh -c nc -z localhost 8095`
2. Check using HTTP GET Request: `httpget path: /health-status`
3. Check using TCP: `tcpSocket Port: 8095`

## 69. Create Kubernetes Liveness & Readiness Probes

Create Liveness Probe with Command

- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - nc -z localhost 8095
  initialDelaySeconds: 60
  periodSeconds: 10
```

Create Readiness Probe with HTTP GET.

- When applying this, you'll see that we have to wait for atleast 60s while it is in `READY 0/1`. After `60->90` seconds, it should now be in `READY 1/1`.

```yaml
readinessProbe:
  httpGet:
    path: /usermgmt/health-status
    port: 8095
  initialDelaySeconds: 60
  periodSeconds: 10
```

Create k8s objects and test:

- User Management Microservice pod will not be in READY state to accept traffic until it completes the initialDelaySeconds=60seconds.

```shell
# create all objects
kubectl apply -f 69-probes/

# list pods
kubectl get pods

# watch list pods screen
kubectl get pdos -w

# describe pod & discuss about init container
kubectl describe pod <usermgmt-microservice-xxxx>

# access application health status page
http://<WorkerNode-Public-IP>:31231/usermgmt/health-status
```

Clean-Up:

```yaml
# Delete All
kubectl delete -f 69-probes/

# List Pods
kubectl get pods

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```

## 70. Kubernetes Resources - Requests & Limits

- We can specify how much each container and a pod needs in terms of resources like CPU & Memory
- When we provide this information in our pod, the scheduler uses this information to decide which node to place the Pod on.
- When you specify a resource limit for a Container, the kubelet enforces those `limits` so that the running container is not allowed to use more of that resource than the limit you set.
- The kubelet also reserves at least the `request` amount of that system resource specifically for that container to use.

### Add Requests & Limits

Update the deployments file to add requests:

- **Limits** aim to set the maximum amount of compute resources that a container can consume. It helps prevent a single container from using more than its fair share of resources, which could degrade the performance of other containers on the same node. Limits act as a hard cap on resources such as CPU and memory.
- **Requests** specify the minimum resources a container needs to start and run. This value is used by the Kubernetes scheduler to make informed decisions about where to place pods within the cluster. Nodes must have at least the requested amount of resources free to be eligible to host the pod.

```yaml
resources:
  requests:
    memory: "128Mi" # 128 MebiByte is equal to 135 Megabyte (MB)
    cpu: "500m" # `m` means milliCPU
  limits:
    memory: "500Mi"
    cpu: "1000m" # 1000m is equal to 1 vCPU
```

### Create k8s objects & Test

Apply the manifests:

```shell
# Create All Objects
kubectl apply -f 70-resources

# List Pods
kubectl get pods

# Watch List Pods screen
kubectl get pods -w

# Describe Pod & Discuss about init container
kubectl describe pod <usermgmt-microservice-xxxxxx>

# Access Application Health Status Page
http://<WorkerNode-Public-IP>:31231/usermgmt/health-status

# List Nodes & Describe Node
kubectl get nodes
kubectl describe node <Node-Name>
```

### Clean-Up

Delete all k8s objects created as part of this section:

```shell
# Delete All
kubectl delete -f 70-resources/

# List Pods
kubectl get pods

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```

## 71. Kubernetes Namespaces - Introduction

References:

- https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/
- https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/
- https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/

Introduction

- Namespaces are also called _Virtual clusters_ in our _physical_ k8s cluster
- We use this in environments where we have _many users spread_ across multiple teams or projects
- Clusters with _tens of users_ ideally don't need to use namespaces

Benefits:

- Creates _isolation boundary_ from other k8s objects
- We can limit the resources like _CPU_, _Memory_ on per namespace basis (`Resource Quota`)

You can get namespaces via:

```shell
$ kubectl get namespace
$ kubectl get ns

default           Active   16h # is created by default, when no NS is mentioned, pods are sent here
kube-node-lease   Active   16h
kube-public       Active   16h # is created by default, and is visible by everyone including un-authed users
kube-system       Active   16h # all k8s related objects are created in kube-system
```

- `default`: The general-purpose namespace for objects that do not specify a namespace. It is the primary workspace for users and applications if no other namespace is designated.
- `kube-system`: is reserved for objects created by the Kubernetes system itself, such as the API server, scheduler, and DNS services.
- `kube-public`: This namespace is readable by all clients, including unauthenticated ones. It is typically used for storing cluster-wide, publicly visible data, such as a ConfigMap containing cluster information.
- `kube-node-leas`e: This namespace holds Lease objects, one per node, which allow the kubelet to send heartbeats. This mechanism helps the control plane detect node failures efficiently and improves cluster scalability.

## 72. Kubernetes Namespaces - Create Imperatively using kubectl

- Namespaces allow to split-up resources into different groups.
- Resource names should be unique in a namespace
- We can use namespaces to create multiple environments like dev, staging, and production etc.
- K8s will alwawsy list the resources from `default` namespace unless we provide exclusively from which namespace we need information from.

### Namespaces Generic - Deploy in `dev1` and `dev2`

Create namespace

- Copy over the yaml files from section 69 (probes). Do not add the resource limits section as we can run out of resources in this section. Remember to comment out nodeports as well.

```shell
# list namespaces
kubectl get ns

# create namespace
kubectl create namespace <namespace-name>
kubectl create namespace dev1
kubectl create namespace dev2

# list namespaces
kubectl get ns
```

NodePort in UserMgmt redundant ports via file `07-usermanagement-service.yaml`:

- Whenever we create with same manifests multiple environments like `dev1`, `dev2`, with namespaces, we cannot have same worker with multiple services.
- We will have port conflict
- Its good for k8s system to provide dynamic nodeport for us in such situations.

```yaml
nodePort: 31231
```

Error if not commented:

```yaml
The Service "usermgmt-restapp-service" is invalid: spec.ports[0].nodePort: Invalid value: 31231: provided port is already allocated
```

Deploy all k8s objects:

```shell
# deploy all k8s objects
kubectl apply -f 72-ns-imperative -n dev1
kubectl apply -f 72-ns-imperative -n dev2

# List all objects from dev1 & dev2 namespaces
kubectl get all -n dev1
kubectl get all -n dev2
```

Verify SC, PVC & PV:

- PVC is a namespace specific resource
- PV and SC are generic (not specific to a namespace)
- **Observation-1**: `Persistent Volume Claim (PVC)` gets created in respective namespaces

```shell
kubectl get pvc -n dev1
kubectl get pvc -n dev2
```

- **Observation-2**: `Storage Class (SC) and Persistent Volume (PV)` gets created generic. No sepcific namespace for them.

```shell
# list sc, pv
kubectl get sc,pv
```

### Access Application

Dev1 Namespace:

```shell
# Get Public IP
kubectl get nodes -o wide

# Get NodePort for dev1 usermgmt service
kubectl get svc -n dev1

# Access Application
http://<Worker-Node-Public-Ip>:<Dev1-NodePort>/usermgmt/health-stauts
```

Dev2 Namespace:

```shell
# Get Public IP
kubectl get nodes -o wide

# Get NodePort for dev2 usermgmt service
kubectl get svc -n dev2

# Access Application
http://<Worker-Node-Public-Ip>:<Dev1-NodePort>/usermgmt/health-stauts
```

### Clean-Up

```shell
# delete namespaces for dev1 & dev2
# this will also delete every resource in this namespace
kubectl delete ns dev1
kubectl delete ns dev2

# list all objects from dev1 & dev2 namespaces
kubectl get all -n dev1
kubectl get all -n dev2

# list namespaces
kubectl get ns

# delete generic items (not specific to namespaces)
# list sc, pv
kubectl get sc,pv

# delete storage class
kubectl delete sc ebs-sc

# get all from all namespaces
kubectl get all --all-namespaces
```

## 73. Kubernetes Namespaces - Limit Range - Introduction

Introduction

- By default, containers run with **unbounded** _compute resources_ on a cluster. Using `ResourceQuota`s, admins (aka _cluster operators_) can restrict consumption and creation of cluster resources (CPU time, Memory, persistent storage, etc) within a specified _namespace_.
- Within a namespace, a `Pod` can consume as much CPU and memory as is allowed by the `ResourceQuota`s that apply to that namespace. As a cluster operator, or as a namespace-level administrator, you might also be concerned about making sure that a single object cannot monopolize all available resources within a namespace.
- A `LimitRange` is a policy to constrain the resource allocations (limits and requests) that you can specify for each applicable kind (Pod, or `PersistentVolumeClaim`) in a namespace

In addition:

- We can limit the min/max range of resources that can be used by a pod in a namespace
- We can specify these limits either via a `default` spec or `min/max` spec.
- In both, we can specify the CPU and Memory values that we allow.

References:

- https://kubernetes.io/docs/concepts/policy/limit-range/

A `LimitRange` manifest can look like:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-cpu-mem-limit-range
spec:
  limits:
    - default: # this section defines default limits
        cpu: 500m
      defaultRequest: # this section defines default requests
        cpu: 500m
      max: # max and min define the limit range
        cpu: "1"
      min:
        cpu: 100m
      type: Container
```

A `ResourceQuota` manifest can look like:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-resource-quota
  namespace: dev3
spec:
  limits:
    - default: # think of this as "limit"
        memory: "512Mi" # If not specified, the container's memory limit is set to 512Mi, which is the default memory limit for the namespace
        cpu: "500m" # if not specified, default limit is 1vCPU per container
      defaultRequest: # think of this as "request"
        memory: "256Mi" # If not specified, default it will take from whatever specified in limits.default.memory
        cpu: "300m" # If not specified, default it will take from whatever specified in limits.default.cpu
      type: Container
```

## 74. Kubernetes Namespaces - Create Limit Range k8s manifest

Create Namespace manifest `01-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev3
```

Create LimitRange manifest `02-limit-range.yaml`:

- Instead of specifying resources like `cpu` and `memory` in every container spec of a pod definition, we can provide the default CPU & Memory for all containers in a namespace using `LimitRange`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-cpu-mem-limit-range
  namespace: dev3
spec:
  limits:
    - default:
        memory: "512Mi" # If not specified, the container's memory limit is set to 512Mi, which is the default memory limit for the namespace
        cpu: "500m" # if not specified, default limit is 1vCPU per container
      defaultRequest:
        memory: "256Mi" # If not specified, default it will take from whatever specified in limits.default.memory
        cpu: "300m" # If not specified, default it will take from whatever specified in limits.default.cpu
      type: Container
```

Update all files from 03 to 10 with `namespace: dev3` in top metadata section. You can ignore the `StorageClass` manifest though, as it is generic (not bound to a namespace):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-mysql-pv-claim
  namespace: dev3
```

## 75. Kubernetes Namespaces - Limit Range - Update App k8s Manifest, Deploy

Create k8s objects & test:

```shell
# Create All Objects
kubectl apply -f 74-ns-limits/

# List Pods
kubectl get pods -n dev3 -w

# View Pod Specification (CPU & Memory) for mysql
kubectl get pod <pod-name> -o yaml -n dev3

# Get & Describe Limits
kubectl get limits -n dev3
kubectl describe limits default-cpu-mem-limit-range -n dev3

# Get NodePort
kubectl get svc -n dev3

# Get Public IP of a Worker Node
kubectl get nodes -o wide

# Access Application Health Status Page
http://<WorkerNode-Public-IP>:<NodePort>/usermgmt/health-status
```

Clean-Up:

```shell
kubectl delete -f 74-ns-limits
```

## 76. Kubernetes - Resource Quota

Introduction

- When several users or teams share a cluster with a fixed number of nodes, there is a concern that one team could use more than its fair share of resources. `ResourceQuota`s are a tool for admins to address this.
- A `ResourceQuota` provides constraints that limit aggregate resource consumption per namespace. It can also limit the **quantity of objects that can be created in a namespace** by API kind, as well as the total **infrastructure resources** that may be consumed by API objects found in that namespace

References:

- https://kubernetes.io/docs/concepts/policy/resource-quotas/

Ensure the namespace is created first:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev3
```

Create ResourceQuote manifest `02-resource-quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-resource-quota
  namespace: dev3
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: "2Gi"
    pods: "5"
    configmaps: "5"
    persistentvolumeclaims: "5"
    replicationcontrollers: "5"
    secrets: "5"
    services: "5"
```

Create k8s objects & test:

```shell
# create all objects
kubectl apply -f 76-resource-quota

# list pods
kubectl get pods -n dev3 -w

# view pod specification (cpu & memory)
kubectl get pod <pod-name> -o yaml -n dev3

# get and describe limits
kubectl get quota -n dev3
kubectl describe quota ns-resource-quota -n dev3

# get nodeport
kubectl get svc -n dev3

# get public ip of a worker node
kubectl get nodes -o wide

# access application health status page
# http://<WorkerNode-Public-IP>:<NodePort>/usermgmt/health-status
```

Clean-Up:

```shell
kubectl delete -f 76-resource-quota
```

### References:

- https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/
- https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/

### Additional References:

- https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/
- https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/
