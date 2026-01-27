# Kubernetes Fundamentals - Declarative Approach using YAML

### References

- Pod: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/
- Service: https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/
- Kubernetes API Reference: https://kubernetes.io/docs/reference/kubernetes-api/

## 44. YAML Basics - Introduction

- YAML is not a Markup language
- YAML is used to _store information_ about different things
- We can use YAML to \*define Key, Value pairs8 like variables, lists and objects
- Is very similary to _JSON_ (JavaScript Object Notation)
- Primarily focuses on _readability_ and user _friendliness_
- Is designed to be _clean and easy to read_
- We can define YAML files with two different extensions `abc.yml` and `abc.yaml`.

We are going to cover:

- Comments
- Key Value pairs
- Dictionary or Map
- Array / Lists
- Spaces
- Document Separator

### Comments & Key Value Pairs

Space after colon is mandatory to differentiate between key and value

```yaml
# this is a sample comment
name: karani
age: 23
city: Nairobi
```

### Dictionary / Map

- Is a set of properties grouped together after an item
- Equal amount of blanks pace required for all items under a dictionary

```yaml
person:
  name: karani
  age: 23
  city: Nairobi
```

### Array / Lists

Dash indicates an element of an array

```yaml
person: # dictionary
  name: karani
  age: 23
  city: Nairobi
  hobbies: # list
    - cycling
    - cooking
  others: [cycling, cooking] # list with a different notation
```

### Multiple Lists

Dash indicates an element of an array

```yaml
person: # dictionary
  name: karani
  age: 23
  city: Nairobi
  hobbies: # list
    - cycling
    - cooking
  others: [cycling, cooking] # list with a different notation
  friends: #
    - name: friend1
      age: 22
    - name: friend2
      age: 25
```

### Sample Pod template for reference

```yaml
apiVersion: v1 # String
kind: Pod # String
metadata: # Dictionary
  name: myapp-pod
  labels: # Dictionary
    app: myapp
spec:
  containers: # List
    - name: myapp
      image: stacksimplify/kubenginx:1.0.0
      ports:
        - containerPort: 80
          protocol: "TCP"
        - containerPort: 81
          protocol: "TCP"
```

### Document Separator

You can have two documents in one yaml file using the `---` YAML document separator:

```yaml
person:
  name: karani
  age: 23
  city: Nairobi
___
person:
  name: john
  age: 25
  city: Mombasa
```

## 45. Create Pods with YAML

The basic structure of a K8s YAML file is as follows

```yaml
apiVersion:
kind:
metadata:

spec:
```

### Create simple pod definition using YAML

Create a `02-pod-definition.yml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
    - name: myapp
      image: stacksimplify/kubenginx:1.0.0
      ports:
        - containerPort: 80
```

Apply the file

```shell
kubectl create -f 02-pod-definition.yml
# or
kubectl apply -f 02-pod-definition.yaml

# List pods
kubectl get pods
```

## 46. Create a NodePort Service with YAML and Access Application via Browser

Create a `03-nodeport-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-pod-nodeport-service
spec:
  type: NodePort
  selector: # load balance traffic across pods matching this selector
    app: myapp
  ports: # access traffic sent on port 80
    - name: http
      port: 80 # service port
      targetPort: 80 # container port
      nodePort: 30040 # nodeport
```

Now apply the file:

```shell
# create service
kubectl apply -f 03-nodeport-service.yml

# list service
kubectl get svc

# get public IP
kubectl get nodes -o wide
```

Access the appication using Public IP

```
http://<node1-public-ip>:<node-port>/hello

# for k3d
http://localhost:30040/hello
```

## 47. Create ReplicaSets using YAML

Create a `04-replicaset-definition.yml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp2-rs
spec:
  replicas: 3 # 3 pods should exists at all times
  selector: # pods label should be defined in ReplicaSet label selector
    matchLabels:
      app: myapp2
  template:
    metadata:
      name: myapp2-pod
      labels:
        app: myapp2 # atleast 1 pod label should match with the ReplicaSet label selector
    spec:
      containers:
        - name: myapp2
          image: stacksimplify/kubenginx:2.0.0
          ports:
            - containerPort: 80
```

Apply the file:

```shell
kubectl apply -f 04-replicaset-definition.yml`

# list ReplicaSets
kubectl get rs
```

Delete a pod and verify that the ReplicaSet immediately creates a new pod:

```shell
# list pods
kubectl get pods

# delete pod
kubectl delete pod <pod-name>
```

## 48. Create NodePort Service with YAML and Access Application via Browser

Create NodePort service for ReplicaSet in `05-replicaset-nodeport-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: replicaset-nodeport-service
spec:
  type: NodePort
  selector: # load balance traffic across pods matching this selector
    app: myapp2
  ports: # access traffic sent on port 80
    - name: http
      port: 80 # service port
      targetPort: 80 # container port
      nodePort: 30030 # nodeport
```

Apply the nodeport file:

```shell
kubectl apply -f 05-replicaset-nodeport-service.yml

# list nodeport services
kubectl get svc

# get public ip
kubectl get nodes -o wide

# access the application
http://<Worker-Node-Public-IP>:<NodePort>
http://<Worker-Node-Public-IP>:30030
```

## 49. Create Depolyment with YAML and Test

Create a deployment file `06-deployment-definition.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp3-deployment
spec:
  replicas: 3 # 3 pods should exists at all times
  selector: # pods label should be defined in Deployment label selector
    matchLabels:
      app: myapp3
  template:
    metadata:
      name: myapp3-pod
      labels:
        app: myapp3 # atleast 1 pod label should match with the Deployment label selector
    spec:
      containers:
        - name: myapp3
          image: stacksimplify/kubenginx:3.0.0
          ports:
            - containerPort: 80
```

Create NodePort service for ReplicaSet in `07-deployment-nodeport-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: deployment-nodeport-service
spec:
  type: NodePort
  selector: # load balance traffic across pods matching this selector
    app: myapp3
  ports: # access traffic sent on port 80
    - name: http
      port: 80 # service port
      targetPort: 80 # container port
      nodePort: 30020 # nodeport
```

Apply the files:

```shell
kubectl apply -f 06-deployment-definition.yaml
kbuectl get deploy
kubectl get rs
kubectl get po

# create the nodeport service
kubectl apply -f 07-deployment-nodeport-service.yml

# list service
kubectl get svc

# get public ip
kubectl get nodes -o wide

# access the applicaiton
# http://<worker-node-public-ip>:30020
```

## 50. Backend Application - Create Deployment with YAML and test

We are going to deploy a frontend/backend application and use that to explore:

- NodePort service
- ClusterIP Service

### Create Backend Deployment & Cluster IP Service

Create a deployment file `08-backend-deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-restapp
  labels:
    app: backend-restapp
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-restapp
  template:
    metadata:
      labels:
        app: backend-restapp
        tier: backend
    spec:
      containers:
        - name: backend-restapp
          image: stacksimplify/kube-helloworld:1.0.0
          ports:
            - containerPort: 8080
```

Create a ClusterIP file `09-backend-clusterip-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-backend-service ## VERY VERY IMPORTANT - NGINX PROXYPASS needs this name
  labels:
    app: backend-restapp
    tier: backend
spec:
  #type: Cluster IP is a default service
  selector:
    app: backend-restapp
  ports:
    - name: http
      port: 8080 # ClusterIp Service Port
      targetPort: 8080 # Container Port
```

Apply the files:

```shell
kubectl apply -f 08-backend-deployment.yml -f 09-backend-clusterip-service.yml

kubectl get all
```

## 51. Frontend Application - Create Deployment and NodePort Service

Create a frontend deployment file `10-frontend-deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-nginxapp
  labels:
    app: frontend-nginxapp
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend-nginxapp
  template:
    metadata:
      labels:
        app: frontend-nginxapp
        tier: frontend
    spec:
      containers:
        - name: frontend-nginxapp
          image: stacksimplify/kube-frontend-nginx:1.0.0
          ports:
            - containerPort: 80
```

Create a frontend NodePort service `11-frontend-nodeport-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nginxapp-nodeport-service
  labels:
    app: frontend-nginxapp
    tier: frontend
spec:
  type: NodePort
  selector:
    app: frontend-nginxapp
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30010
```

Apply the files:

```shell
kubectl apply -f 10-frontend-deployment.yml -f 11-frontend-nodeport-service.yml

kubectl get all
```

Access REST Application:

```shell
# Get External IP of nodes using
kubectl get nodes -o wide

# Access REST Application  (Port is static 31234 configured in frontend service template)
http://<node1-public-ip>:30010/hello
```

## 52. Deploy and Test - Frontend and Backend Applications

### Delete & Recreate Objects using kubectl apply

Delete Objects (file by file):

```shell
kubectl delete -f 08-backend-deployment.yml -f 09-backend-clusterip-service.yml -f 10-frontend-deployment.yml -f 11-frontend-nodeport-service.yml

kubectl get all

mkdir 12-fullstack

cp 08-backend-deployment.yml 12-fullstack
cp 09-backend-clusterip-service.yml 12-fullstack
cp 10-frontend-deployment.yml 12-fullstack
cp 11-frontend-nodeport-service.yml 12-fullstack
```

Recreate object using YAML files in a folder:

```shell
kubectl apply -f 12-fullstack

kubectl get all
```

Delete object susing YAML files in folder

```shell
kubectl delete -f 12-fullstack

kubectl get all
```
