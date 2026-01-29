# EKS Hosted Applications Storage with AWS RDS- Relational Database Service

## 77. EKS Storage - RDS DB Introduction

- So far we have been running our MySQL in our EKS Cluster
- In production environments though, this is not good.
- We could try to use `StatefulSets` to help ensure high availability.

Drawbacks of EBS CSI for MySQL DB:

- For a non-DB admin, this might be VERY hard, we are only DevOps admins.
- Complex setup to achieve HA (`StatefulSets`) etc
- Complex Multi-AZ Support for EBS
- Complex Master-Master MySQL setups
- Complex Master-Slave MySQL setups
- No Automatic Backup & Recovery
- No Auto-Upgrade MySQL

We can use a managed database (AWS RDS) to help alleviate these risks.

- AWS Will manage the database for us
- Inside k8s, we can use an `ExternalName` Service to connect to MySQL from k8s.

Advantages of RDS:

- High availability
- Backup & Recovery
- Read Replicas
- Metrics & Monitoring
- Automatic Upgrades
- Multi-AZ Support

Architecture:

- We are in AWS Cloud
- In AWS, we have a VPC provisioned as part of our cluster
- Currently our EKS Cluster exists in a public subnet, in the future, we can have the load balancer here, and move the nodes to the private subnet
- Our MySQL RDS instance is in our private subnet

![eks rds network](../media/09-1-eks-rds-network.png)

## 78. Create RDS DB

Review VPC of our EKS Cluster

- Go to Services -> Networking & Content Delivery -> VPC
- VPC Name: `eksctl-eksdemo1-cluster/VPC`
- Click on the VPC and take note of the names of the private subnets.

Pre-requisite-1: create DB Security Group:

- Go to Services -> Compute -> EC2 -> Network & Security -> Security Groups
- Create security group to allow access for RDS Database on port 3306
- Security group name: `eks_rds_db_sg`
- Description: Allow access for RDS database on port 3306
- VPC: `eksctl-eksdemo1-cluster/VPC`
- Inbound Rules
  - Type: MysQL/Aurora
  - Protocl: TPC
  - Port: 3306
  - Source: Anywhere (0.0.0.0/0)
  - Description: Allow access for RDS Database on port 3306
- Outbound Rules
  - Leave defaults

Pre-requisite-2: create DB Subnet Group in RDS

- Go to Services -> RDS -> Subnet Groups
- Click on _Create DB Subnet Group_
  - Name: `eks-rds-db-subnet-group`
  - Description: EKS RDS DB Subnet Group
  - VPC: `eksctl-eksdemo1-cluster/VPC`
  - Availability Zones: `us-east-1a`, `us-east-1b`
  - Subnets: 2 subnets in 2 AZs (only the private ones for the clsuter VPC)
    - `eksctl-eksdemo1-cluster/SubnetPrivateUSEAST1A`
    - `eksctl-eksdemo1-cluster/SubnetPrivateUSEAST1B`
  - Click on _Create_

Create RDS Database:

- Go to Services -> RDS
- Click on **Create Database**
  - Database Creation Method: Standard Create (Full Configuration)
  - Engine Options: MySQL
  - Edition: MySQL Community
  - Version: default populated
  - Template Size: free tier (sandbox)
  - DB Instance Identifier: `usermgmtdb`
  - Master Username: `dbadmin`
  - Master Password: `dbpassword11`
  - Confirm Password: `dbpassword11`
  - DB Instance Size: leave default
  - Storage: Leave Defaults
  - Connectivity:
    - VPC: `eksctl-eksdemo1-cluster/VPC`
    - Additional Connectivity Configuration:
      - Subnet Group: `eks-rds-db-subnetgroup`
      - Publicly accessible: Yes (for our learning and troubleshooting - if required)
    - VPC Security Group: Choose Existing, remove default.
      - Name: `eks-rds-db-securitygroup`
    - Availability Zone: No Preference (we can choose `us-east-1a`)
    - Database Port: 3306
  - Rest all leave to defaults
- Click on _Create Database_
- Wait and get the connectivity name by clicking on `usermgmtdb` -> `Connectivity & Security` -> `Endpoints`

Edit Database Security to Allow Access from `0.0.0.0/0`:

- Go to EC2 -> Security Groups -> `eks-rds-db-securitygroup`
- Edit Inbound Rules
  - Sources: Anywhere (0.0.0.0/0) (Allow access from everywhere, for now)

Create k8s `externalName` service Manifest and Deploy

- Create mysql `externalName` service `01-mysql-externalname-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: ExternalName
  externalName: usermgmtdb.c2fammqas0yq.us-east-1.rds.amazonaws.com
```

Deploy manifest:

```yaml
kubectl apply -f kube-manifests/01-mysql-externalname-service.yaml
```

Connect to RDS DB using `kubectl` and create `usermgmt` schema/db

```shell
kubectl run -it --rm --image=mysql:latest --restart=Never mysql-client -- mysql -h usermgmtdb.c2fammqas0yq.us-east-1.rds.amazonaws.com -u dbadmin -pdbpassword11

mysql> show schemas;
mysql> create database usermgmt;
mysql> show schemas;
mysql> exit
```

Update `user-management-microservice-deployment.yaml` to change the username from `root` to `dbadmin`:

```yaml
# Change From
- name: DB_USERNAME
  value: "root"

# Change To
- name: DB_USERNAME
  value: "dbadmin"
```

Deploy user management microservice and test:

- Ensure you don't have resource limits on the `03-user-management-deployment.yaml` manifest. Having that block might cause problems as we are running in an underpowered node.

```shell
# Deploy all Manifests
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Stream pod logs to verify DB Connection is successful from SpringBoot Application
kubectl logs -f <pod-name>
```

Access Application:

```yaml
# Capture Worker Node External IP or Public IP
kubectl get nodes -o wide

# Access Application
http://<Worker-Node-Public-Ip>:30030/usermgmt/health-status
```

Clean Up:

- Leave the DB running though, we'll need it when learning load balancers.
- You can modify the DB and disable backups (retention period 0)

```yaml
# Delete all Objects created
kubectl delete -f kube-manifests/

# Verify current Kubernetes Objects
kubectl get all
```
