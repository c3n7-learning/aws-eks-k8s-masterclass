# ALB Ingress - Internal Application Load Balancer

## 125. Introduction to Ingress Internal ALB

- We worked with internet-facing ALBs
- To test out internal load balancers, we'll use the annotation `alb.ingress.kubernetes.io/scheme: internal`
- We **cannot** connect to internal LB directly from the internet to test it.
- We will deploy a **curl pod** in EKS so we can connect to the curl pod in our EKS Cluster and test the **Internal LB Endpoint** using the **curl** command

## 126. Create Internal ALB Using Ingress and Test and Clean Up

Copy over manifests from `17-alb-ingress-ssl-discovery-host/120-ssl-tls`:

```
01-app1-deployment-nodeport.yaml
02-app2-deployment-nodeport.yaml
03-app3-deployment-nodeport.yaml
04-alb-ingress.yaml
```

Update `04-alb-ingress.yaml`:

```yaml
# Annotations Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-internal-lb-demo
  labels:
    app: ingress-internal-lb-demo
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: lb-ingress-internal
    # Ingress Core
    alb.ingress.kubernetes.io/scheme: internal
    # Health Check Settings
    # If you have multiple targets in the ingress, move this health check to the NodePort service
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/success-codes: "200"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"
spec:
  ingressClassName: my-aws-ingress-class # Ingress class
  defaultBackend:
    service:
      name: app3-nginx-nodeport-service
      port:
        number: 80
  rules:
    - http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-nodeport-service
                port:
                  number: 80
    - http:
        paths:
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: app2-nginx-nodeport-service
                port:
                  number: 80
```

Create the curl pod `05-curl-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
spec:
  containers:
    - name: curl
      image: curlimages/curl
      command: ["sleep", "86400"]
```

Deploy all Application Kubernetes Manifests and Verify:

```shell
# Deploy kube-manifests
kubectl apply -f 126-ingress-internal/

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

How to test this Internal Load Balancer?

- We are going to deploy a `curl-pod` in EKS Cluster
- We connect to that `curl-pod` in EKS Cluster and test using `curl commands` for our sample applications load balanced using this Internal Application Load Balancer

Verify Internal LB:

```shell
# Will open up a terminal session into the container
kubectl exec -it curl-pod -- sh

# We can now curl external addresses or internal services:
curl https://google.com/
curl <INTERNAL-INGRESS-LB-DNS>

# Default Backend Curl Test
curl internal-lb-ingress-internal-1126314747.us-east-1.elb.amazonaws.com

# App1 Curl Test
curl internal-lb-ingress-internal-1126314747.us-east-1.elb.amazonaws.com/app1/index.html

# App2 Curl Test
curl internal-lb-ingress-internal-1126314747.us-east-1.elb.amazonaws.com/app2/index.html

# App3 Curl Test
curl internal-lb-ingress-internal-1126314747.us-east-1.elb.amazonaws.com
```

Clean Up:

```shell
# Delete Manifests
kubectl delete -f 126-ingress-internal
```
