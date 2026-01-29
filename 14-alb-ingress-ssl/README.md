# ALB Ingress - SSL & SSL Redirect using AWS Application Load Balancer

## 105. Introduction to ALB Ingress SSL

We are going to introduce:

- AWS Route 53
- AWS Certificate Manager

We will perform three tasks:

1. Register AWS Route53 DNS Domain
2. Create SSL Certificate in AWS Certificate manager
3. Update SSL Annotations in Ingress Service

Reference:

- https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/08-NEW-ELB-Application-LoadBalancers/08-04-ALB-Ingress-SSL
- [AWS Load Balancer Controller Annotation Reference](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)

You need to register a domain on Route53 to explore this section.

## 106. Register Domain AWS Route53

Go to

1. Services -> Networking & Content Delivery -> Route53
2. Choose domain name `timothykarani.com` -> Continue
3. Contact Details

- Enter personal info
- Enable privacy protection

4. Review:

- Do you want to renew domain? yes
- Accept Terms and Conditions

5. Create Domain

## 107. Create SSL Certificate in AWS Certificate Manager

Pre-requisite: You should have a registered domain in Route53

- Go to Services -> Security, Identity, & Compliance -> Certificate Manager -> Create a Certificate
- Click on **Request a Certificate**
  - Choose the type of certificate for ACM to provide: Request a public certificate
  - Add domain names: `*.yourdomain.com` (in my case it is going to be `*.timothykarani.com`)
  - Select a Validation Method: **DNS Validation**
  - Click on **Confirm & Request**
- **Validation**
  - Click on **Create record in Route 53**
- Wait for 5 to 10 minutes and check the **Validation Status**

## 108. Update SSL Ingress Annotation, Deploy

Copy over the files from the last section:

```
01-app1-deployment-nodeport.yaml
02-app2-deployment-nodeport.yaml
03-app3-deployment-nodeport.yaml
04-alb-ingress-context-path.yaml
```

Update `04-alb-ingress-context-path.yaml`:

- `metadata.name`: `ingress-ssl-demo`
- `alb.ingress.kubernetes.io/load-balancer-name`: `ssl-ingress`

```yaml
## SSL Settings
alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:662513131574:certificate/ba79d928-9650-49a8-acc5-6d7d5f1840f6
#alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-1-2017-01 #Optional (Picks default if not used)
```

Deploy and verify:

```shell
# Deploy kube-manifests
kubectl apply -f 107-ssl/

# Verify Ingress Resource
kubectl get ingress

# Verify Apps
kubectl get deploy
kubectl get pods

# Verify NodePort Services
kubectl get svc
```

Verify Load Balancer & Target Groups in the AWS Console:

- Load Balancer - Listeners (Verify both 80 & 443)
- Load Balancer - Rules (Verify both 80 & 443 listeners)
- Target Groups - Group Details (Verify Health check path)
- Target Groups - Targets (Verify all 3 targets are healthy)

Add DNS in Route53

- Go to **Services -> Route 53**
- Go to **Hosted Zones**
  - Click on **yourdomain.com** (in my case timothykarani.com)
- Create a **Record Set**
  - **Name:** ssldemo101.timothykarani.com
  - **Alias:** yes
  - **Region** `us-east-1`
  - **Alias Target:** Copy our ALB DNS Name here (Sample: ssl-ingress-551932098.us-east-1.elb.amazonaws.com)
  - Click on **Create**

Access Application using newly registered DNS Name

```
# HTTP URLs
http://ssldemo101.timothykarani.com/app1/index.html
http://ssldemo101.timothykarani.com/app2/index.html
http://ssldemo101.timothykarani.com/

# HTTPS URLs
https://ssldemo101.timothykarani.com/app1/index.html
https://ssldemo101.timothykarani.com/app2/index.html
https://ssldemo101.timothykarani.com/
```

## 109. Update SSL Ingress Redirection Annotation, Deploy, Test and CleanUp

Reference:

- https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/08-NEW-ELB-Application-LoadBalancers/08-05-ALB-Ingress-SSL-Redirect

Update `04-alb-ingress-context-path.yaml`:

```yaml
# SSL Redirect Setting
alb.ingress.kubernetes.io/ssl-redirect: "443"
```

Deploy and verify:

```shell
# Deploy kube-manifests
kubectl apply -f 108-ssl-redirect/

# Verify Ingress Resource
kubectl get ingress

# Verify Apps
kubectl get deploy
kubectl get pods

# Verify NodePort Services
kubectl get svc
```

Verify Load Balancer & Target Groups in the AWS Console:

- Load Balancer - Listeners (Verify both 80 & 443)
- Load Balancer - Rules (Verify both 80 & 443 listeners)
- Target Groups - Group Details (Verify Health check path)
- Target Groups - Targets (Verify all 3 targets are healthy)
- Listner (port 80) - only 1 rule

Access Application using newly registered DNS Name

```
# HTTP URLs
http://ssldemo101.timothykarani.com/app1/index.html
http://ssldemo101.timothykarani.com/app2/index.html
http://ssldemo101.timothykarani.com/

# HTTPS URLs
https://ssldemo101.timothykarani.com/app1/index.html
https://ssldemo101.timothykarani.com/app2/index.html
https://ssldemo101.timothykarani.com/
```

Clean-Up:

```shell
# Delete Manifests
kubectl delete -f kube-manifests/

## Delete Route53 Record Set
- Delete Route53 Record we created (ssldemo101.stacksimplify.com)
```
