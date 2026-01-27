# EKS Hosted Applications Storage with AWS EBS - Elastic Block Store

References:

- https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/04-EKS-Storage-with-EBS-ElasticBlockStore

## 56. EKS Storage Introduction

We have various types of EKS Storage:

1. In-Tree EBS Provisioner - Legacy & will be deprecated soon.

Container Storage Interface.

- Latest and greatest available today & in Beta release & ready for production use
- As of today, _not supported_ on AWS EKS Fargate (Serveless)
- Allows EKS Clusters to _manage lifecycle_ of EBS Volumes for persistent storage, EFS file system & FSx for Luster File Systems.
- EBS & EFS - Supported for k8s 1.14 & later
- FSx for Luster File - Supported for k8s 1.16 & later

2. EBS CSI Driver
3. EFS CSI Driver
4. FSx for Luster CSI (windows stuff and more)

### Elastic Block Store

- Provides _block level storage volumes_ for use with _EC2 & Container instances_.
- We can mount these _volumes as devices_ on our EC2 & Container instances.
- EBS volumes that are attached to an instance are _exposed as storage volumes that persist independently_ from the life of the EC2 or Container instance.
  - Even if our EC2/container instances are terminated, the EBS volumes are maintained
- We can _dynamically change_ the configuration of a volume attached to an instance
- AWS recommends EBS for data that must be _quickly accessible_ and requires _long-term persistence_.
- EBS is well suited to both _database-style applications_ that rely on random reads and writes, and to _throughput-intensive applications_ that perform long, continuous reads & writes.

We are going to build:

1. A MySQL Deployment
2. We'll leverage an **AWS Elastic Block Store - EBS** in this deployment.
   - We will leverage:
     - Storage Class
     - Persistent Volume Claim
     - Config Map
     - Environment Variables
     - Volumes
     - Volume Mounts
     - Deployment
     - ClusterIP Service
3. We'll deploy a deployment with a UserMgmt microservice. We'll leverage:
   - NodePort Service
   - Deployment
   - Environment Variables

## 57. Install EBS CSI Driver

### Pod Identity Agent:

You should:

1. Open _EKS Console_ -> _Clusters_ -> select your cluster (`eksdemo1`)
2. Go to _Add-ons_ -> _Get more add-ons_
3. Search for _EKS Pod Identity Agent_
4. Click _Next_ -> _Create_

This install a `DaemonSet` (`eks-pod-identity-agent`) that enables Pod identity associations.

```shell
# list k8s PIA resources
kubectl get daemonset -n kube-system
kubectl get ds -n kube-system

# list k8s pods
kubectl get pods -n kube-system
```

### EBS CSI Driver:

Go to:

1. EKS -> Clusters -> `eksdemo1` -> Add Ons tab -> Get More Addons
2. Select `Amazon EBS CSI Driver` -> Next
3. Add-on access: `EKS Pod Identity`
4. Click **Create Recommended Role**:
   - Trusted entity:
     - Entity Type: `AWS Service`
     - Service or Use Case: `EKS`
     - Use Case: `EKS - Pod Identity`
   - Permissions Policies: `AmazonEBSCSIDriverPolicy` and `AmazonEKSClusterPolicy` will be already selected. Click next
   - Review -> Permissions policy summary: ensure `AmazonEBSCSIDriverPolicy` and `AmazonEKSClusterPolicy` are selected`
   - Click _Create Role_
   - You should see that `AmazonEKSPodIdentityAmazonEBSCSIDriverRole` has been craeted. Go back to Add-Ons
5. Refresh `Pod Identity IAM role for service account` and select `AmazonEKSPodIdentityAmazonEBSCSIDriverRole`
6. Click **Next** -> Click **Create**.

### Verify Installation

List pods in `kube-system`

```shell
kubectl get pods -n kube-system
```

Expected output (sample):

```
NAME                                  READY   STATUS    RESTARTS   AGE
aws-node-59xkd                        2/2     Running   0          84m
coredns-6b9575c64c-6m4jf              1/1     Running   0          95m
ebs-csi-controller-59d564ccff-bj655   6/6     Running   0          49s
ebs-csi-node-nkmbs                    3/3     Running   0          49s
eks-pod-identity-agent-c9drq          1/1     Running   0          3m12s
kube-proxy-qfgkc                      1/1     Running   0          84m
metrics-server-7cbd59cb7c-vh2j4       1/1     Running   0          93m
```

Verify DaemonSets:

```shell
kubectl get ds -n kube-system
```

Expected output (sample):

```
NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR              AGE
aws-node                 2         2         2       2            2           <none>                     94m
ebs-csi-node             2         2         2       2            2           kubernetes.io/os=linux     25s
ebs-csi-node-windows     0         0         0       0            0           kubernetes.io/os=windows   25s
eks-pod-identity-agent   2         2         2       2            2           <none>                     2m48s
kube-proxy               2         2         2       2            2           <none>                     94m
```

Verify Deployments:

```shell
kubectl get deploy -n kube-system
```

Expected output (sample):

```
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
coredns              2/2     2            2           26m
ebs-csi-controller   0/2     2            0           2m54s
metrics-server       2/2     2            2           24m
```

## 58. Create Kubernetes Manifests for Storage Class, PVC and ConfigMap

We are going to create a MySQL Database with persistence storage using AWS EBS Volumnes

Storage Class `01-storage-class.yaml`:

- `WaitForFirstConsumer` will delay the volume binding and provisioning of a persistent volume until a Pod using the `PersistentVolumeClaim` is created.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

Persistent Volume Claim `02-persistent-volume-claim.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi
```

User Management Config Map `03-user-management-configmap.yaml`:

- We are going to create a `usermgmt` database schema during the mysql pod creation time which we will leverage when we deploy User Management Microservice.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: usermanagement-dbcreation-script
data:
  mysql_usermgmt.sql: |-
    DROP DATABASE IF EXISTS usermgmt;
    CREATE DATABASE usermgmt;
```

## 59. Create Kubernetes Manifests for MySQL Deployment & ClusterIP Service

MySQL deployment `04-mysql-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.6
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: dbpassword11
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
            - name: usermanagement-dbcreation-script
              mountPath: /docker-entrypoint-initdb.d # https://hub.docker.com/_/mysql Refer Initializing a fresh instance
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: ebs-mysql-pv-claim
        - name: usermanagement-dbcreation-script
          configMap:
            name: usermanagement-dbcreation-script
```

MySQL ClusterIP Service `05-mysql-clusterip-service.yaml`:

- At any point of time we are going to have only one mysql pod in this desing so `ClusterIP: None` will use the `Pod IP Address` isntead of creating or allocating a separate IP for `MySQL Cluster IP Service`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
  clusterIP: None # This means we are goin to use Pod IP
```

### Create the k8s manifests

Create Storage Class manifest

```shell
# Create StorageClass, PVC and Database
kubectl apply -f kube-manifests/

# List Storage Classes
kubectl get sc

# List PVC
kubectl get pvc

# List PV
kubectl get pv

# list pods
kubectl get pods

# List pods based on label name
kubectl get pods -l app=mysql
```

## 60. Test by connecting to MySQL Database

Connecting to MySQL Database

```shell
# connect to MySQL Database
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -pdbpassword11

# or use mysql client latest tag
kubectl run -it --rm --image=mysql:latest --restart=Never mysql-client -- mysql -h mysql -pdbpassword11

# verify usermgmt schema got created which we provided in ConfigMap
mysql> show schemas;
```

## 61. Storage References:

References

- Few features are still in alpha stage as on today (Example:Resizing), but once they reach beta you can start leveraging those templates and make your trials.
- **EBS CSI Driver:** https://github.com/kubernetes-sigs/aws-ebs-csi-driver
- **EBS CSI Driver Dynamic Provisioning:** https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes/dynamic-provisioning
- **EBS CSI Driver - Other Examples like Resizing, Snapshot etc:** https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes
- **k8s API Reference Doc:** https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#storageclass-v1-storage-k8s-io

## 62. Create Kubernetes Manifests for User Management Microservice Deployment

Reference:

- https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/
  04-EKS-Storage-with-EBS-ElasticBlockStore/04-03-UserManagement-MicroService-with-MySQLDB

Introduction

- We are going to deploy a User Management MicroService which will connect to MySQL DB Schema

Create a deployment file `06-user-management-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: usermgmt-microservice
  labels:
    app: usermgmt-restapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: usermgmt-restapp
  template:
    metadata:
      labels:
        app: usermgmt-restapp
    spec:
      containers:
        - name: usermgmt-restapp
          image: stacksimplify/kube-usermanagement-microservice:1.0.0
          ports:
            - containerPort: 8095
          env:
            - name: DB_HOSTNAME
              value: "mysql"
            - name: DB_PORT
              value: "3306"
            - name: DB_NAME
              value: "usermgmt"
            - name: DB_USERNAME
              value: "root"
            - name: DB_PASSWORD
              value: "dbpassword11"
```

Create a NodePort service `07-user-management-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: usermgmt-restapp-service
  labels:
    app: usermgmt-restapp
spec:
  type: NodePort
  selector:
    app: usermgmt-restapp
  ports:
    - port: 8095
      targetPort: 8095
      nodePort: 30030
```

## 63. Test User Management Microservice with MySQL Database in Kubernetes

Ensure you have deleted everything from the previous sub-section, to demonstrate something:

```shell
kubectl delete -f kube-manifests
```

Apply the manifests:

```shell
kubectl apply -f kube-manifests/

# list pods
kubectl get pods

# verify logs of usermgmt microservice pod
kubectl logs -f <pod-name>

# verify sc, pvc, pv
kubectl get sc, pvc, pv
```

Observed Problems:

- If we deploy all manifests at the same time, by the time mysql is ready, our `User Management Microservice` will be restarting multiple times due to unavailability of the DB. `kubectl get pods`
- To avoid such situations, we can apply `initContainers` concept to our microservice's `Deployment Manifest`.
- We will see that in our next section, but for now, lets continue to test the application

Access Application (remember to allow public external http access via `remoteAccess` security group):

```shell
kubectl get svc

# Get Public IP
kubectl get nodes -o wide

# Access Health Status API for User Management Service
http://<EKS-WorkerNode-Public-IP>:30030/usermgmt/health-status
```

## 64: Test User Management Microservice UMS using Postman

Test User Management Microservice using Postman

### Download Postman client

- https://www.postman.com/downloads/

### Import Project to Postman

- Import the postman project `AWS-EKS-Masterclass-Microservices.postman_collection.json` present in folder `04-03-UserManagement-MicroService-with-MySQLDB`

### Create Environment in postman

- Go to Settings -> Click on Add
- **Environment Name:** UMS-NodePort
  - **Variable:** url
  - **Initial Value:** http://WorkerNode-Public-IP:31231
  - **Current Value:** http://WorkerNode-Public-IP:31231
  - Click on **Add**

### Test User Management Services

- Select the environment before calling any API
- **Health Status API**
  - URL: `{{url}}/usermgmt/health-status`
- **Create User Service**
  - URL: `{{url}}/usermgmt/user`
  - `url` variable will replaced from environment we selected
  - Ignore any error related to notifications, we'll check on that later

```json
{
  "username": "admin1",
  "email": "dkalyanreddy@gmail.com",
  "role": "ROLE_ADMIN",
  "enabled": true,
  "firstname": "fname1",
  "lastname": "lname1",
  "password": "Pass@123"
}
```

- **List User Service**
  - URL: `{{url}}/usermgmt/users`

- **Update User Service**
  - URL: `{{url}}/usermgmt/user`

```json
{
  "username": "admin1",
  "email": "dkalyanreddy@gmail.com",
  "role": "ROLE_ADMIN",
  "enabled": true,
  "firstname": "fname2",
  "lastname": "lname2",
  "password": "Pass@123"
}
```

- **Delete User Service**
  - URL: `{{url}}/usermgmt/user/admin1`

### Verify Users in MySQL Database

```
# Connect to MYSQL Database
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -u root -pdbpassword11

# Verify usermgmt schema got created which we provided in ConfigMap
mysql> show schemas;
mysql> use usermgmt;
mysql> show tables;
mysql> select * from users;
```

### Clean-Up

- Delete all k8s objects created as part of this section

```
# Delete All
kubectl delete -f kube-manifests/

# List Pods
kubectl get pods

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```
