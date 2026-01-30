# ALB Ingress SSL Discovery Host and TLS

## 118. Introduction to Ingress SSL Discovery

### SSL Discovery - Host

- TLS certificates for ALB listeners can be automatically discovered with **hostnames** from ingress resources if the `alb.ingress.kubernetes.io/certificate-arn` annotation is not specified.
- They are discovered and attached **implicitly**=
- The controller will attempt to discover **TLS certificates** from the **tls field** in Ingress and **host field** in Ingress rules.

> **Note**  
> You need to explicitly specify to use HTTPS listener with listen-ports annotation:  
> `alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'`

### SSL Discovery using tls

Example ingress config

```yaml
spec
# ...
  tls
  - hosts:
    - "*.timothykarani.com"
# ...
```

- If we either go the _hosts_ or _tls_ routes, we don't have to explicitly specify the certificate through the `alb.ingress.kubernetes.io/certificate-arn` annotation.

## 119. Implement SSL Discovery Host Demo

Reference

- https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/08-NEW-ELB-Application-LoadBalancers/08-10-Ingress-SSL-Discovery-host

Copy over the manifests from `16-alb-ingress-name-based-host/117-ingress-nvr`:

```
01-app1-deployment-nodeport.yaml
02-app2-deployment-nodeport.yaml
03-app3-deployment-nodeport.yaml
04-alb-ingress-context-path.yaml
```

Update `04-alb-ingress.yaml`:

```yaml
# Annotations Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-certdiscoveryhost-demo
  labels:
    app: ingress-certdiscoveryhost-demo
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: certdiscoveryhost-ingress
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
    # alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:662513131574:certificate/ba79d928-9650-49a8-acc5-6d7d5f1840f6
    #alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-1-2017-01 #Optional (Picks default if not used)
    # SSL Redirect Setting
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    # External DNS - For creating a Record Set in Route53
    external-dns.alpha.kubernetes.io/hostname: default102.timothykarani.com
spec:
  ingressClassName: my-aws-ingress-class # Ingress class
  defaultBackend:
    service:
      name: app3-nginx-nodeport-service
      port:
        number: 80
  rules:
    - host: app102.timothykarani.com
      http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-nodeport-service
                port:
                  number: 80
    - host: app202.timothykarani.com
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
kubectl apply -f 119-ssl-host/

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

Verify Route53:

- Go to Services -> Route53
- You should see **Record Sets** added for
  - default102.timothykarani.com
  - app102.timothykarani.com
  - app202.timothykarani.com

Perform nslookup tests before accessing Application:

- Test if our new DNS entries registered and resolving to an IP Address

```shell
# nslookup commands
nslookup default102.timothykarani.com
nslookup app102.timothykarani.com
nslookup app202.timothykarani.com
```

Positive Case: Access Application using DNS domain:

```shell
# Access App1
http://app102.timothykarani.com/app1/index.html

# Access App2
http://app202.timothykarani.com/app2/index.html

# Access Default App (App3)
http://default102.timothykarani.com
```

Clean Up:

```shell
# Delete Manifests
kubectl delete -f 119-ssl-host/

## Verify Route53 Record Set to ensure our DNS records got deleted
- Go to Route53 -> Hosted Zones -> Records
- The below records should be deleted automatically
  - default102.timothykarani.com
  - app102.timothykarani.com
  - app202.timothykarani.com
```

References:

- https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/cert_discovery/

## 120. Implement SSL Discovery TLS Demo

Reference:

- https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/08-NEW-ELB-Application-LoadBalancers/08-11-Ingress-SSL-Discovery-tls

We will

- Automatically discover the SSL Certificate from AWS Certificate manager Service using `spec.tls.host`

Copy over the files from the previous sub-section, then update `04-alb-ingress.yaml`:

```yaml
# Annotations Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-certdiscoverytls-demo
  labels:
    app: ingress-certdiscoverytls-demo
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: certdiscoverytls-ingress
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
    # alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:662513131574:certificate/ba79d928-9650-49a8-acc5-6d7d5f1840f6
    #alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-1-2017-01 #Optional (Picks default if not used)
    # SSL Redirect Setting
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    # External DNS - For creating a Record Set in Route53
    external-dns.alpha.kubernetes.io/hostname: default103.timothykarani.com
spec:
  ingressClassName: my-aws-ingress-class # Ingress class
  defaultBackend:
    service:
      name: app3-nginx-nodeport-service
      port:
        number: 80
  tls:
    - hosts:
        - "*.timothykarani.com"
  rules:
    - host: app103.timothykarani.com
      http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-nodeport-service
                port:
                  number: 80
    - host: app203.timothykarani.com
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
kubectl apply -f 120-ssl-tls/

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

Verify Route53:

- Go to Services -> Route53
- You should see **Record Sets** added for
  - default102.timothykarani.com
  - app102.timothykarani.com
  - app202.timothykarani.com

Perform nslookup tests before accessing Application:

- Test if our new DNS entries registered and resolving to an IP Address

```shell
# nslookup commands
nslookup default103.timothykarani.com
nslookup app103.timothykarani.com
nslookup app203.timothykarani.com
```

Positive Case: Access Application using DNS domain:

```shell
# Access App1
http://app103.timothykarani.com/app1/index.html

# Access App2
http://app203.timothykarani.com/app2/index.html

# Access Default App (App3)
http://default103.timothykarani.com
```

Clean Up:

```shell
# Delete Manifests
kubectl delete -f 120-ssl-tls/

## Verify Route53 Record Set to ensure our DNS records got deleted
- Go to Route53 -> Hosted Zones -> Records
- The below records should be deleted automatically
  - default103.timothykarani.com
  - app103.timothykarani.com
  - app203.timothykarani.com
```
