# EKS Pod Identity

References:

- https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/04-EKS-Storage-with-EBS-ElasticBlockStore/04-00-EKS-Pod-Identity-Agent

## 53. Introduction to EKS Pod Identity Agent

- In AWS EKS, how will your Pod access other AWS services like S3/ElastiCache/RDS
- PIA: Pod Identity Agent.

We are going to:

- Create a pod with `aws-cli` in it
- We will try to access `s3` from the `aws-cli pod`
- We will then create an IAM Role with `s3 ReadOnly` IAM Permission
- We will then install the `eks-pod-identity-agent-DaemonSet`: this DaemonSet will be installed in each of our k8s cluster nodes.
- We will then create a service account for our `aws-cli`: `aws-cli Service Account`
- We will then create an `EKS Pod Identity Association`
- We will then restart the pod, and try to access `s3` from the `aws-cli pod`

```shell
aws s3 list
```

This applies to other AWS services like:

- AWS S3 bucket
- EBS Volumes
- DynamoDB

### High Level Overview

![pod identity workflow](../media/06-pod-identity-workflow.jpg)

1. **Create IAM Role**:
   - An IAM Administrator creates a role that can be assumed by the new EKS service principal: `pods.eks.amazonaws.com`:
     - Trust policy can be restricted by cluster ARN or AWS account
     - Attach required IAM policies (e.g., `AmazonS3ReadOnlyAccess`)

2. **Create Pod Identity Association**:
   - The EKS administrator associates the IAM Role with a k8s service account + namespace.
   - Done via the EKS Console or `CreatePodIdentityAssociation` API.

3. **Webhook Mutation**:
   - When a pod using that Service Account is created, the **EKS Pod Identity Webhook** (running in the control plane) mutates the pod spec:
     - Injects environment variables such as `AWS_CONTAINER_CREDENTIALS_FULL_URI`.
     - Mounts a projected service account token for use by the Pod Identity Agent.

**Verify Mutation Example**:

```shell
$ kubectl exec -it aws-cli --env | grep "AWS_CONTAINER"

AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token
```

4. **Pod Requests Credentials** inside the pod, then AWS SDK/CLI uses the default credential provider chain
   - It discovers the injected environment variables and calls the **EKS Pod Identity Agent** running as a DaemonSet on the worker node.
   - 4a. **PIA Agent Calls EKS Auth API**:
     - The Pod Identity Agent exchanges the projected token with the **EKS Auth API** using `AssumeRoleForPodIdentity`
   - 4b. **EKS Auth API Validates Association**:
     - The API checks the Pod Identity Association (Namespace + ServiceAccount -> IAM Role).
     - If valid, it returns temporary IAM credentials back to the Pod Identity Agent

5. **Pod Accesses AWS Resources** - The AWS SDK/CLI inside the pod now as valid, short-lived credentials and can call AWS services (e.g. list S3 buckets).

Key Notes:

- Pods receive temporary IAM credentials automatically -- no `aws configure` needed.
- Leverages the **standard AWS credential chain** (no code changes required).
- Requires the **EKS Pod Identity Agent Add-on** running on worker nodes.
- Supported only with newer versions of AWS SDKs and CLI

## 54. PIA Demo Part-1

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

### Deploy AWS CLI Pod (without Pod Identity Association)

Create service account `01-service-account.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-cli-sa
  namespace: default
```

Create a simple k8s pod with AWS CLI Image `02-aws-cli-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  namespace: default
spec:
  serviceAccountName: aws-cli-sa
  containers:
    - name: aws-cli
      image: amazon/aws-cli
      command: ["sleep", "infinity"]
```

Deploy CLI Pod

```shell
kubectl apply -f 01-service-account.yaml -f 02-aws-cli-pod.yaml
kubectl get pods

kubectl get serviceaccount
kubectl get sa
```

Exec into the pod and try to list s3 buckets:

```shell
kubectl exec -it aws-cli -- aws s3 ls
```

You'll see the error:

```
An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:sts::662513131574:assumed-role/eksctl-eksdemo1-nodegroup-eksdemo1-NodeInstanceRole-1EnJ44LoT6TQ/i-0bb97227618c080a6 is not authorized to perform: s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action
command terminated with exit code 254
```

## 55. PIA Demo Part-2

### Create IAM Role for Pod Identity

Go to

1. _IAM Console_ -> _Roles_ -> _Create Role_
   - Service or Use case: `EKS`
   - Use case: `EKS - Pod Identity`
2. Permissions Policies: Attach `AmazonS3ReadOnlyAccess` policy
3. Create role -> example name: `EKS-PodIdentity-S3-Role-101`

### Create Pod Identity Association

- Go to EKS Console -> Cluster -> Access -> Pod Identity Associations
- Create new association
  - Namespace: `default`
  - Service Account: `aws-cli-sa`
  - IAM Role: `EKS-PodIdentity-S3-Role-101`
  - Click on **create**

### Test Again

Pods don't automatically refresh credentials after a new Pod Identity Association, they must be restarted

- Restart Pod

```shell
# delete pod
kubectl delete pod aws-cli -n default

# create pod
kubectl apply -f 02-aws-cli-pod.yaml

# list pods
kubectl get pods
```

Exec into the pod and try to list s3 buckets:

```shell
kubectl exec -it aws-cli -- aws s3 ls
```

### Clean Up

- Delete the `aws-cli` pod

```shell
kubectl delete -f 01-service-account.yaml -f 02-aws-cli-pod.yaml
```

- Remove Pod Identity Association → via _EKS Console_ → _Access_ → _Pod Identity Associations_
- Remove IAM role → via _IAM Console_ → _Roles_ `EKS-PodIdentity-S3-ReadOnly-Role-101`
