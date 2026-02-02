# Extra - EKS Cloudflare External DNS

Reference:

- https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/cloudflare/#cloudflare-dns

## 90.1 Pre-Requisites

#### EKS Cluster

Create eks cluster and worker nodes (if not created):

```shell
# Create Cluster (Section-01-02)
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup


# Get List of clusters (Section-01-02)
eksctl get cluster

# Template (Section-01-02)
eksctl utils associate-iam-oidc-provider \
    --region region-code \
    --cluster <cluster-name> \
    --approve

# Replace with region & cluster name (Section-01-02)
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve

# Create EKS NodeGroup in VPC Private Subnets (Section-07-01)
eksctl create nodegroup --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-private1 \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking
```

Verify cluster, node groups and configure `kubectl` cli if not configured:

```shell
# verify eks cluster
eksctl get cluster

# verify eks node groups
eksctl get nodegroup --cluster=eksdemo1

# verify if any IAM Service Accounts present in EKS Cluster
eksctl get iamserviceaccount --cluster=eksdemo1
# No iamserviceaccounts found

# configure kubeconfig for kubectl
eksctl get cluster # to get cluster name
aws eks --region <region-code> update-kubeconfig --name <cluster-name>
aws eks --region us-east-1 update-kubeconfig --name eksdemo1

# verify eks nodes in eks cluster using kubectl
kubectl get nodes

# verify using AWS Management Console
# 1. EKS EC2 Nodes (verify subnet in networking tab)
# 2. EKS Cluster
```

#### Application Load Balancer Controller

Create IAM policy for the AWS Load Balancer Controller that allows it to make calls to AWS APIs on your behalf:

- Only do this if you don't have the role, otherwise just copy the arn from the AWS Management Console

```shell
# delete files before download
rm iam_policy_latest.json

# download IAM latest Policy
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/docs/install/iam_policy.json

## Download specific version
curl -o iam_policy_v2.3.1.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.1/docs/install/iam_policy.json

# create IAM Policy using policy downloaded
aws iam create-policy \
        --policy-name AwsLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy_latest.json

## Sample Output
$ aws iam create-policy \
>     --policy-name AWSLoadBalancerControllerIAMPolicy \
>     --policy-document file://iam_policy_latest.json
{
    "Policy": {
        "PolicyName": "AwsLoadBalancerControllerIAMPolicy",
        "PolicyId": "ANPAZUQG4EA3KOGZJO2XV",
        "Arn": "arn:aws:iam::662513131574:policy/AwsLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2026-01-29T11:04:57+00:00",
        "UpdateDate": "2026-01-29T11:04:57+00:00"
    }
}
```

Create IAM Role using eksctl

```shell
# verify if there's any existing service account
kubectl get sa -n kube-system
kubectl get sa aws-load-balancer-controller -n kube-system
# Observation: nothing with name `aws-load-balancer-controller` should exist

# Template
eksctl create iamserviceaccount \
              --cluster=eksdemo1 \
              --namespace=kube-system \
              --name=aws-load-balancer-controller \
              --attach-policy-arn=arn:aws:iam::662513131574:policy/AwsLoadBalancerControllerIAMPolicy \
              --override-existing-serviceaccounts \
              --approve
```

Verify using `eksctl` cli:

```shell
# get IAM Service Account
eksctl get iamserviceaccount --cluster eksdemo1

# sample output
NAMESPACE       NAME                            ROLE ARN
kube-system     aws-load-balancer-controller    arn:aws:iam::662513131574:role/eksctl-eksdemo1-addon-iamserviceaccount-kube--Role1-W8uBn8azq18c
```

Verify k8s service account using `kubectl`:

```shell
# verify if any existing service account exists
kubectl get sa -n kube-system
kubectl get sa aws-load-balancer-controller -n kube-system
# observation: we should see a new service account created

kubectl describe sa aws-load-balancer-controller -n kube-system
```

Install AWS Load Balancer Controller using HELM:

```shell
# add the eks-charts repository
helm repo add eks https://aws.github.io/eks-charts

# update your local repo to make sure you have the most recent charts
helm repo update

# install the AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
             -n kube-system \
             --set clusterName=eksdemo1 \
             --set serviceAccount.create=false \
             --set serviceAccount.name=aws-load-balancer-controller \
             --set region=us-east-1 \
             --set vpcId=vpc-vpc-0af275aca53da9c62 \
             --set image.repository=public.ecr.aws/eks/aws-load-balancer-controller
```

Verify that the controller is installed:

Run:

```shell
kubectl -n kube-system get deployment
kubectl -n kube-system get deployment aws-load-balancer-controller
kubectl -n kube-system describe deployment aws-load-balancer-controller
```

Sample Output:

```shell
$ kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           27s
```

Verify AWS Load Balancer Controller Webhook service created:

```shell
kubectl -n kube-system get svc
kubectl -n kube-system get svc aws-load-balancer-webhook-service
kubectl -n kube-system describe svc aws-load-balancer-webhook-service
```

Sample output:

```shell
$ kubectl -n kube-system get svc aws-load-balancer-webhook-service
NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
aws-load-balancer-webhook-service   ClusterIP   10.100.53.52   <none>        443/TCP   61m
```

Verify labels in service and selector labels in deployment:

```shell
kubectl -n kube-system get svc aws-load-balancer-webhook-service -o yaml
kubectl -n kube-system get deployment aws-load-balancer-controller -o yaml
```

Deploy the Ingress Class Resource:

We can have ingresses such as:

- AWS **EKS** ALB Ingress Controller
- **NGINX** Ingress Controller
- **AKS** Application Gateway Ingress Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: my-aws-ingress-class
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb
```

Apply it:

```shell
kubectl apply -f 02-ingress-class
```

## 90.2 External DNS

Create a secret file with Cloudflare API Token `01-helm/1-cloudflare-secret.yaml`:

- [Ensure you have a Cloudflare API Token](https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/cloudflare/#creating-cloudflare-credentials)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-key
type: Opaque
data:
  # Output from echo -n 'my-token' | base64
  # abcdefg
  apiKey: abcdefgBase64
```

Apply the manifest:

```yaml
kubectl apply -f 01-helm/1-cloudflare-secret.yaml
```

The create a `01-helm/2-cloudflare-helm-values.yaml` file that we'll use to deploy external-dns with cloudflare:

```yaml
provider:
  name: cloudflare
policy: sync
env:
  - name: CF_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: cloudflare-api-key
        key: apiKey
```

Deploy the manifest:

```shell
# install external-dns
helm repo add --force-update external-dns https://kubernetes-sigs.github.io/external-dns/

helm upgrade --install external-dns external-dns/external-dns --values 01-helm/2-cloudflare-helm-values.yaml

# list all resources from default namespace
kubectl get all

# list pods (external-dns pod should be in running state)
kubectl get pods

# verify deployment by checking logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

### Test out an nginx deployment

Create a `01-nginx-deployment.yaml` file:

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
    # https://kubernetes-sigs.github.io/aws-load-balancer-controller/v3.0/guide/service/annotations/#traffic-routing
    service.beta.kubernetes.io/aws-load-balancer-type: external

    # https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/cloudflare/#deploying-an-nginx-service
    external-dns.alpha.kubernetes.io/hostname: demo201.karanimwenda.com
    external-dns.alpha.kubernetes.io/ttl: "300" #optional
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
spec:
  type: LoadBalancer
  selector:
    app: app1-nginx
  ports:
    - port: 80
      targetPort: 80
```

Apply the manifests:

```yaml
kubectl apply -f 03-kubenginx
```

## 90.3 TLS/SSL

Reference:

- Provisioning a certificate on ACM: `14-alb-ingress-ssl/README.md`

The base deployments/nodeports have been copied over:

- Refer to `17-alb-ingress-ssl-discovery-host/120-ssl-tls`

Create an ingress manifest `04-alb-ingress.yaml` file:

```yaml
# Annotations Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-cloudflare-acm-demo
  labels:
    app: ingress-cloudflare-acm-demo
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: cloudflare-acm-ingress
    # Ingress Core
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Health Check Settings
    # If you have multiple targets in the ingress, move this health check to the NodePort service
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/success-codes: "200"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"
    ## SSL Settings
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:662513131574:certificate/dde97232-17ce-4580-9824-44618e4b6775
    #alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-1-2017-01 #Optional (Picks default if not used)
    # SSL Redirect Setting
    alb.ingress.kubernetes.io/ssl-redirect: "443"

    # External DNS - For creating a Record Set in Cloudflare
    # https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/cloudflare/#deploying-an-nginx-service
    external-dns.alpha.kubernetes.io/hostname: default403.karanimwenda.com
    external-dns.alpha.kubernetes.io/ttl: "300" #optional
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
spec:
  ingressClassName: my-aws-ingress-class # Ingress class
  defaultBackend:
    service:
      name: app3-nginx-nodeport-service
      port:
        number: 80
  rules:
    - host: app401.karanimwenda.com
      http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-nodeport-service
                port:
                  number: 80
    - host: app402.karanimwenda.com
      http:
        paths:
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: app2-nginx-nodeport-service
                port:
                  number: 80
```

Apply the manifests:

```shell
# Deploy kube-manifests
kubectl apply -f 04-cloudflare-acm/

# Verify Ingress Resource
kubectl get ingress

# Verify Apps
kubectl get deploy
kubectl get pods

# Verify NodePort Services
kubectl get svc
```

Verify Load Balancer & Target Groups:

- Load Balancer - Listeneres (Verify both 80 & 443)
- Load Balancer - Rules (Verify both 80 & 443 listeners)
- Target Groups - Group Details (Verify Health check path)
- Target Groups - Targets (Verify all 3 targets are healthy)
- **PRIMARILY VERIFY - CERTIFICATE ASSOCIATED TO APPLICATION LOAD BALANCER**

Verify External DNS Log:

```shell
# Verify External DNS logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

Verify Cloudflare DNS:

- Go to Cloudflare -> `karanimwenda.com` -> DNS
- You should see **Record Sets** added for
  - default403.karanimwenda.com
  - app401.karanimwenda.com
  - app402.karanimwenda.com

Perform nslookup tests before accessing Application:

- Test if our new DNS entries registered and resolving to an IP Address

```shell
# nslookup commands
nslookup default403.karanimwenda.com
nslookup app401.karanimwenda.com
nslookup app402.karanimwenda.com
```

Positive Case: Access Application using DNS domain:

```shell
# Access App1
https://app401.karanimwenda.com/app1/index.html

# Access App2
http://app402.karanimwenda.com/app2/index.html

# Access Default App (App3)
http://default403.karanimwenda.com
```

Clean Up:

```shell
# Delete Manifests
kubectl delete -f 04-cloudflare-acm/

## Verify Route53 Record Set to ensure our DNS records got deleted
- Go to Route53 -> Hosted Zones -> Records
- The below records should be deleted automatically
  - default403.karanimwenda.com
  - app401.karanimwenda.com
  - app402.karanimwenda.com
```
