# Microservices Deployments on EKS

## 169. Microservices Introduction

- **Microservices** - aka **microservice architecture** -is an architectural style that structures an application as a **collection of services** that are
  - Highly maintainable and testable
  - Loosely coupled
  - Independently deployable
  - Organized around business capabilities
  - Owned by a small team

Benefits:

- **Developer independence**: Small teams work in parallel and can iterate **faster** than large teams.
- **Isolation and resilience**: If a **component dies**, you spin up another while and the rest of the application continues to function.
- **Scalability**: Smaller components take up fewer resources and can be scaled to meet **increasing demand** for that component only.
- **Lifecycle automation**: individual components are easier to fit into **continuous delivery pipelines** and complex deployment scenarios not possible with monoliths.
- **Relationship to the business:** Microservice architectures are split along business domain boundaries, **increasing independence** and understanding across the organization.

## 170. Microservices Deployment on EKS - Introduction

We are going to have 2 microservices

- User Management Microservice (Users DB)
  - Create User API
  - List Users API
  - Delete User API
  - Health Status API

Notification Microservice (SMTP Server)

- Send Notification API (on create user from previous api)
- Health Status API

![microservice architecture](../media/25-1-microservice-architecture.png)

## 171. Pre-requisite Checks

List of Docker Images used in this section
| Application Name | Docker Image Name |
| ------------------------------- | --------------------------------------------- |
| User Management Microservice | stacksimplify/kube-usermanagement-microservice:1.0.0 |
| Notifications Microservice V1 | stacksimplify/kube-notifications-microservice:1.0.0 |
| Notifications Microservice V2 | stacksimplify/kube-notifications-microservice:2.0.0 |

### Pre-requisite 1: AWS RDS Database, ALB Ingress Controller & External DNS

#### AWS RDS Database

- We have created AWS RDS Database as part of section [09-eks-hosted-app-storage](../09-eks-hosted-app-storage/README.md)
- We even created a `externalName service: 01-MySQL-externalName-Service.yml` in our Kubernetes manifests to point to that RDS Database.

#### ALB Ingress Controller & External DNS

- We are going to deploy a application which will also have a `ALB Ingress Service` and also will register its DNS name in Route53 using `External DNS`
- Which means we should have both related pods running in our EKS cluster.
- We have installed **ALB Ingress Controller** as part of section [alb ingress install](../11-alb-ingress-install-aws-elb/README.md)
- We have installed **External DNS** as part of section [external dns installation](../15-alb-ingress-external-dns/README.md)

```
# Verify alb-ingress-controller pod running in namespace kube-system
kubectl get pods -n kube-system

# Verify external-dns pod running in default namespace
kubectl get pods
```

### Pre-requisite-2: Create Simple Email Service - SES SMTP Credentials

#### SMTP Credentials

- Go to Services -> Business Applications -> Simple Email Service
- SMTP Settings -> Create My SMTP Credentials
- **IAM User Name:** append the default generated name with microservice or something so we have a reference of this IAM user created for our ECS Microservice deployment
- Download the credentials and update the same for below environment variables which you are going to provide in kubernetes manifest `04-NotificationMicroservice-Deployment.yml`

```
AWS_MAIL_SERVER_HOST=email-smtp.us-east-1.amazonaws.com
AWS_MAIL_SERVER_USERNAME=****
AWS_MAIL_SERVER_PASSWORD=***
AWS_MAIL_SERVER_FROM_ADDRESS= use-a-valid-email@gmail.com
```

- **Important Note:** Environment variable AWS_MAIL_SERVER_FROM_ADDRESS value should be a **valid** email address and also verified in SES.

#### Verify Email Addresses to which notifications we need to send.

- We need two email addresses for testing Notification Service.
- SES -> **Identities** -> **Create Identity**
  - Verify a New Email Address
  - Email Address Verification Request will be sent to that address, click on link to verify your email.
  - **From Address:** stacksimplify@gmail.com (replace with your ids during verification)
  - **To Address:** dkalyanreddy@gmail.com (replace with your ids during verification)
- **Important Note:** We need to ensure all the emails (FromAddress email) and (ToAddress emails) to be verified here.
  - Reference Link: https://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-email-addresses.html
- Environment Variables
  - AWS_MAIL_SERVER_HOST=email-smtp.us-east-1.amazonaws.com
  - AWS_MAIL_SERVER_USERNAME=**\***
  - AWS_MAIL_SERVER_PASSWORD=**\***
  - AWS_MAIL_SERVER_FROM_ADDRESS=stacksimplify@gmail.com

## 172. Review Notification Microservice Kubernetes Manifests

Copy over the files from the upstream repo:

- https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/12-Microservices-Deployment-on-EKS/kube-manifests

```
01-mysql-externalname-service.yml
02-user-management-deployment.yml
03-user-management-nodeport.yml
04-notification-microservice-deployment.yml
05-smtp-externalname-service.yml
06-notification-microservice-clusterip.yml
07-alb-ingress.yml
```

### Create Notification Microservice Deployment Manifest

- Update environment Variables for Notification Microservice
- **Notification Microservice Deployment**

```yml
- name: AWS_MAIL_SERVER_HOST
  value: "smtp-service"
- name: AWS_MAIL_SERVER_USERNAME
  value: "AKIABCDEDFASUBKLDOAX"
- name: AWS_MAIL_SERVER_PASSWORD
  value: "Bdsdsadsd32qcsads65B4oLo7kMgmKZqhJtEipuE5unLx"
- name: AWS_MAIL_SERVER_FROM_ADDRESS
  value: "stacksimplify@gmail.com"
```

## 173. Review User Management Microservice Kubernetes Manifests

Update User Management Microservice Deployment Manifest with Notification Service Environment Variables.

- User Management Service new environment varibales related to Notification Microservice in addition to already which were configured related to MySQL
- Update in `02-user-management-deployment.yml`

```yml
- name: NOTIFICATION_SERVICE_HOST
  value: "notification-clusterip-service"
- name: NOTIFICATION_SERVICE_PORT
  value: "8096"
```

Update ALB Ingress Service Kubernetes Manifest:

- Update Ingress Service to ensure only target it is going to have is User Management Service
- Remove /app1, /app2 contexts

```yml
    external-dns.alpha.kubernetes.io/hostname: services.timothykarani.com, ums.timothykarani.com
spec:
  ingressClassName: my-aws-ingress-class # Ingress Class
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: usermgmt-restapp-nodeport-service
                port:
                  number: 8095
```

## 174. UMS & NS Microservices Deployment & Test

Deploy Microservices manifests:

```shell
# Deploy Microservices manifests
kubectl apply -f kube-manifests/
```

Verify the Deployment using kubectl:

```shell
# List Pods
kubectl get pods

# User Management Microservice Logs
kubectl logs -f $(kubectl get po | egrep -o 'usermgmt-microservice-[A-Za-z0-9-]+')

# Notification Microservice Logs
kubectl logs -f $(kubectl get po | egrep -o 'notification-microservice-[A-Za-z0-9-]+')

# External DNS Logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns-[A-Za-z0-9-]+')

# List Ingress
kubectl get ingress
```

Verify Microservices health-status via browser:

```
# User Management Service Health-Status
https://services.timothykarani.com/usermgmt/health-status

# Notification Microservice Health-Status via User Management
https://services.timothykarani.com/usermgmt/notification-health-status
https://services.timothykarani.com/usermgmt/notification-service-info
```

Import postman project to Postman client on our desktop:

- Import postman project
- Add environment url
  - https://services.timothykarani.com (**Replace with your ALB DNS registered url on your environment**)

### Test both Microservices using Postman

#### User Management Service

- **Create User**
  - Verify the email id to confirm account creation email received.
- **List User**
  - Verify if newly created user got listed.

## 175. Microservices Rollout new Deployments and CleanUp

### Rollout New Deployment - Set Image Option

```shell
# Rollout New Deployment using Set Image
kubectl set image deployment/notification-microservice notification-service=stacksimplify/kube-notifications-microservice:2.0.0 --record=true

# Verify Rollout Status
kubectl rollout status deployment/notification-microservice

# Verify ReplicaSets
kubectl get rs

# Verify Rollout History
kubectl rollout history deployment/notification-microservice

# Access Application (Should see V2)
https://services.timothykarani.com/usermgmt/notification-health-status

# Roll back to Previous Version
kubectl rollout undo deployment/notification-microservice

# Access Application (Should see V1)
https://services.timothykarani.com/usermgmt/notification-health-status
```

### Rollout New Deployment - kubectl Edit

```shell
# Rollout New Deployment using kubectl edit, change image version to 2.0.0
kubectl edit deployment/notification-microservice

# Verify Rollout Status
kubectl rollout status deployment/notification-microservice

# Verify ReplicaSets
kubectl get rs

# Verify Rollout History
kubectl rollout history deployment/notification-microservice

# Access Application (Should see V2)
https://services.timothykarani.com/usermgmt/notification-health-status

# Roll back to Previous Version
kubectl rollout undo deployment/notification-microservice

# Access Application (Should see V1)
https://services.timothykarani.com/usermgmt/notification-health-status
```

### Rollout New Deployment - Update manifest & kubectl apply

```shell
# Rollout New Deployment by updating yaml manifest 2.0.0
kubectl apply -f kube-manifests/

# Verify Rollout Status
kubectl rollout status deployment/notification-microservice

# Verify ReplicaSets
kubectl get rs

# Verify Rollout History
kubectl rollout history deployment/notification-microservice

# Access Application (Should see V2)
https://services.timothykarani.com/usermgmt/notification-health-status

# Roll back to Previous Version
kubectl rollout undo deployment/notification-microservice

# Access Application (Should see V1)
https://services.timothykarani.com/usermgmt/notification-health-status
```

### Clean-up

```shell
kubectl delete -f kube-manifests/
```
