# ALB Ingress - Target Type - IP Mode

## 123: Introuduction to Ingress Target Type IP Mode

`target-type` specifies how to route traffic to pods (`alb.ingress.kubernetes.io/target-type`)

- It can be Instance or IP mode
- **Instance mode** will route traffic to all ec2 instances within the cluster on the `NodePort` opened for your service.
  - The service must be of type `NodePort` or `LoadBalancer` to use Instance mode
  - The default target type is **Instance Mode** which means there's **no need** to define this annotation in Ingress.
  - `alb.ingress.kubernetes.io/target-type: instance`
- **IP Mode** will route traffic **directly** to the **pod IP**.
  - Is required for **sticky sessions** to work with Application Load Balancers.
  - Fargate only works with this
  - You ned to explicitly define the annotation as `target-type: ip`
  - `alb.ingress.kubernetes.io/target-type: ip`

## 124. Implement Ingress Target Type IP Mode

Copy over manifests from `17-alb-ingress-ssl-discovery-host/120-ssl-tls`:

```
01-app1-deployment-nodeport.yaml
02-app2-deployment-nodeport.yaml
03-app3-deployment-nodeport.yaml
04-alb-ingress.yaml
```

Rename them to:

```
01-app1-deployment-clusterip.yaml
02-app2-deployment-clusterip.yaml
03-app3-deployment-clusterip.yaml
04-alb-ingress.yaml
```

Update `04-alb-ingress.yaml` to:

```yaml
# Annotations Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-target-type-ip-demo
  labels:
    app: ingress-target-type-ip-demo
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: target-type-ip-ingress
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
    external-dns.alpha.kubernetes.io/hostname: demo501.timothykarani.com
    # Target type IP
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: my-aws-ingress-class # Ingress class
  defaultBackend:
    service:
      name: app3-nginx-clusterip-service
      port:
        number: 80
  tls:
    - hosts:
        - "*.timothykarani.com"
  rules:
    - http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-clusterip-service
                port:
                  number: 80
    - http:
        paths:
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: app2-nginx-clusterip-service
                port:
                  number: 80
```

Update the services in app1 to app3 to look like:

- We don't have to use ClusterIP to demonstrate this, but we want to show that IP mode can work without nodeports.
- With IP Mode, traffic goes directly to the Pod

```yaml
#...
metadata:
  name: app1-nginx-clusterip-service
#...
spec:
  type: ClusterIP
#...
```

Deploy all Application Kubernetes Manifests and Verify

```shell
# Deploy kube-manifests
kubectl apply -f 124-ingress-ip-mode/

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
- **PRIMARILY VERIFY - TARGET GROUPS which contain the POD IPs instead of WORKER NODE IP with NODE PORTS**

```shell
# List Pods and their IPs
kubectl get pods -o wide
```

Verify External DNS Log:

```shell
# Verify External DNS logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

Verify Route53:

- Go to Services -> Route53
- You should see **Record Sets** added for
  - demo-501.timothykarani.com

Perform nslookup tests before accessing Application:

- Test if our new DNS entries registered and resolving to an IP Address

```shell
# nslookup commands
nslookup demo-501.timothykarani.com
```

Access Application using DNS domain:

```shell
# Access App1
http://demo-501.timothykarani.com/app1/index.html

# Access App2
http://demo-501.timothykarani.com/app2/index.html

# Access Default App (App3)
http://demo-501.timothykarani.com
```

Clean Up:

```shell
# Delete Manifests
kubectl delete -f 124-ingress-ip-mode/
```

Verify Route53 Record Set to ensure our DNS records got deleted

- Go to Route53 -> Hosted Zones -> Records
- The below records should be deleted automatically
  - demo-501.timothykarani.com
