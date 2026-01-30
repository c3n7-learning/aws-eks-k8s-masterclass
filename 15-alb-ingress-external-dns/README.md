# ALB Ingress - External DNS Install and Implement Ingress & Servic

## 110. Introduction to ALB Ingress External DNS Install

- In previous SSL demos, we added AWS Route53 DNS Records **manually** (ssldemo101.timothykarani.com)
- With External DNS, we can **automatically** add it for a k8s `Ingress` Service or k8s `Service` by defining it as an annotation
- **External DNS**: Used for updating Route53 RecordSets from k8s
- We need to create an IAM Policy, k8s service account, & IAM Role and associate them together for external-dns pod to add or remove entries in AWS Route53 Hosted Zones
- Update External-DNS default manifest to support our needs
- Deploy & Verify logs

![external dns](../media/14-1-external-dns.png)

## 111. Create IAM Policy, k8s Service Account, IAM Role and Verify

This IAM Policy will allow external-dns pod to add, remove DNS entries (Record Sets in a Hosted Zone) in AWS Route53 service

- Go to Services -> Security, Identity, & Compliance -> IAM -> Policies -> Create Policy
  - Click on **JSON** Tab and copy paste below JSON
  - Click on **Visual editor** tab to validate
  - Click on **Review Policy**
  - **Name:** `AllowExternalDNSUpdates`
  - **Description:** Allow access to Route53 Resources for ExternalDNS
  - Click on **Create Policy**
  - https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#iam-policy
  - https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/aws/#iam-policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResources"
      ],
      "Resource": ["arn:aws:route53:::hostedzone/*"]
    },
    {
      "Effect": "Allow",
      "Action": ["route53:ListHostedZones"],
      "Resource": ["*"]
    }
  ]
}
```

Make a note of Policy ARN which we will use in next step:

```
# Policy ARN
arn:aws:iam::662513131574:policy/AllowExternalDNSUpdates
```

### Create IAM Role, k8s Service Account & Associate IAM Policy

- We are going to create a k8s service account named `external-dns` an also a AWS IAM role and associate them by annotating the role ARN in Service Account..
- We are also going to associate the AWS IAM Policy `AllowExternalDNSUpdates` to the newly created AWS IAM Role

Create a pod identity association.

- Prerequisites:
  - Open the `eksdemo` cluster
  - Ensure you have added the `EKS Pod Identity Agent` addon to the cluster

```shell
# template
eksctl create podidentityassociation \
  --cluster <cluster-name> \
  --namespace <service_account_namespace> \
  --service-account-name external-dns \
  --role-name external-dns-pod-identity-role \
  --permission-policy-arns <policy-arn> \
  --approve

# replace cluster name, namespace, service acc and role-arn
# use the role arn from previous step
eksctl create podidentityassociation \
              --cluster eksdemo1 \
              --namespace default \
              --service-account-name external-dns \
              --role-name external-dns-pod-identity-role \
              --permission-policy-arns arn:aws:iam::662513131574:policy/AllowExternalDNSUpdates
```

If not comfortable using CLI, you can use AWS Management Console

- Open IAM -> Roles -> Create
  - Select trusted entity
    - Trusted Entity Type: AWS Service
    - Service or Use Case: EKS
    - Use Case: EKS Pod Identity
  - Add Permissions:
    - Select: `AllowExternalDNSUpdates`
  - Review & Create:
    - Name: `AWSEKSPodIdentityExternalDns`
    - Click create
- Open EKS -> `eksdemo1` -> Access Tab -> Pod Identity Associations -> Create Pod Identity Association
  - IAM Role: `AWSPodIdentityExternalDns`
  - Namespace: `default`
  - Kubernetes Service Account: `external-dns`
  - Copy the ARN: `arn:aws:iam::662513131574:role/AWSEKSPodIdentityExternalDns`

Verify IAM Role & IAM Policy

- Once in the **IAM Role**, verify in **Permissions** tab we have a policy names `AllowExternalDNSUpdates`
- Now make a note of that role ARN, we'll need to update the `external-dns` k8s manifest

```shell
# make a note of role arn
arn:aws:iam::662513131574:role/AWSEKSPodIdentityExternalDns
```

## 112. Review and Update External DNS k8s manifest

Update External DNS k8s manifest `111-helm/external-dns-values.yaml`:

- Original template: https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/aws/#when-using-clusters-with-rbac-enabled
- External DNS: https://artifacthub.io/packages/helm/external-dns/external-dns

```yaml
provider:
  name: aws
serviceAccount:
  eks.amazonaws.com/role-arn: arn:aws:iam::662513131574:role/AWSPodIdentityExternalDns
policy: sync
env:
  - name: AWS_DEFAULT_REGION
    value: us-east-1 # change to region where EKS is installed
```

## 113. Deploy External DNS and Verify Logs

Deploy the manifest:

```shell
# install external-dns
helm repo add --force-update external-dns https://kubernetes-sigs.github.io/external-dns/
helm upgrade --install external-dns external-dns/external-dns --values 111-helm/external-dns-values.yaml

# list all resources from default namespace
kubectl get all

# list pods (external-dns pod should be in running state)
kubectl get pods

# verify deployment by checking logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

References

- https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/aws/#aws
- https://vettom.pages.dev/K8s/external-dns/

## 114. Ingress Service Demo with External DNS

### Update ingress manifest and add ExternalDNS Annotation

- Add annotation with two DNS names
  - `dnstest901.timothykarani.com`
  - `dnstest902.timothykarani.com`

Copy over the files from `14-alb-ingress-ssl/108-ssl-redirect` then update `04-alb-ingress-context-path.yaml`:

```yaml
# External DNS - For creating a Record Set in Route53
external-dns.alpha.kubernetes.io/hostname: dnstest901.timothykarani.com, dnstest902.timothykarani.com
```

### Deploy all k8s manifests and verify

Deploy the manifests

```shell
kubectl apply -f 114-ingress-dns

# verify ingress resource
kubectl get ingress

# verify apps
kubectl get deploy
kubect get pods

# verify nodeport services
kubectl get svc
```

Verify Load Balancer & Target Groups

- Load Balancer - Listeneres (Verify both 80 & 443)
- Load Balancer - Rules (Verify both 80 & 443 listeners)
- Target Groups - Group Details (Verify Health check path)
- Target Groups - Targets (Verify all 3 targets are healthy)

Verify External DNS Log

```shell
# Verify External DNS logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

Perform nslookup tests before accessing Application

- Test if our new DNS entries registered and resolving to an IP Address

```shell
# nslookup commands
nslookup dnstest901.timothykarani.com
nslookup dnstest902.timothykarani.com
```

Access Application using dnstest domains

```t
# HTTP URLs (Should Redirect to HTTPS)
http://dnstest901.timothykarani.com/app1/index.html
http://dnstest901.timothykarani.com/app2/index.html
http://dnstest901.timothykarani.com/

# HTTP URLs (Should Redirect to HTTPS)
http://dnstest902.timothykarani.com/app1/index.html
http://dnstest902.timothykarani.com/app2/index.html
http://dnstest902.timothykarani.com/
```

### Clean Up

Delete Manifests:

```shell
kubectl delete -f kube-manifests/
```

Verify Route53 Record Set to ensure our DNS records got deleted

- Go to Route53 -> Hosted Zones -> Records
- The below records should be deleted automatically
  - dnstest901.timothykarani.com
  - dnstest902.timothykarani.com

## 115. Kubernetes Service Demo with External DNS

Create a deployment file: `01-nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-nginx-deployment
  labels:
    app: app1-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1-nginx
  template:
    metadata:
      labels:
        app: app1-nginx
    spec:
      containers:
        - name: app1-nginx
          image: stacksimplify/kube-nginxapp1:1.0.0
          ports:
            - containerPort: 80
```

Create a `02-loadbalancer-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1-nginx-loadbalancer-service
  labels:
    app: app1-nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: demo101.timothykarani.com
    # https://kubernetes-sigs.github.io/aws-load-balancer-controller/v3.0/guide/service/annotations/#traffic-routing
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: app1-nginx
  ports:
    - port: 80
      targetPort: 80
```

Deploy & Verify:

```shell
# deploy kube-manifests
kubectl apply -f 115-external-dns

# verify apps
kubectl get deploy
kubectl get pods

# verify service
kubectl get svc
```

Verify Load Balancer:

- Go to EC2 -> Load Balancers -> Verify Load Balancer Settings

Verify External DNS Log:

```shell
# Verify External DNS logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

Verify Route53:

- Go to Services -> Route53
- You should see **Record Sets** added for `demo101.timothykarani.com`

Perform nslookup tests before accessing Application:

- Test if our new DNS entries registered and resolving to an IP Address

```shell
# nslookup commands
nslookup demo101.timothykarani.com
```

Access Application using DNS domain:

```shell
# HTTP URL
http://demo101.timothykarani.com/app1/index.html
```

Clean Up:

```shell
# Delete Manifests
kubectl delete -f 115-external-dns
```

Verify Route53 Record Set to ensure our DNS records got deleted

- Go to Route53 -> Hosted Zones -> Records
- The below records should be deleted automatically
  - demo101.timothykarani.com

References

- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/alb-ingress.md
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md
