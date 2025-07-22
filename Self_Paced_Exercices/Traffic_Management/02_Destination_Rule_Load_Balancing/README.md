# Description

Adding a simple Virtual Service with a simple destination route that has the Load Balancer type specified.

# Setup

# Setup

```shell
kubectl create ns istio-system
```

```shell
helm install istio-base istio/base --version="1.26.2" -n istio-system --wait
```

```shell
helm install istiod istio/istiod --version="1.26.2" -n istio-system  --wait
```

```shell
kubectl create ns istio-ingress
```

```shell
helm install istio-ingress istio/gateway --version="1.26.2" -n istio-ingress --wait
```

```shell
kubectl create -f - << EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: default
  namespace: istio-ingress
spec:
  selector:
    app: istio-ingress
    istio: ingress
  servers:
  - port:
      number: 80
      name: http2
      protocol: HTTP2
    hosts:
    - "*"
EOF
```

## Create a backend deployment and service

### Nginx

```shell
kubectl create -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
    service: backend
spec:
  replicas: 1
  selector:
    matchLabels:
        app: nginx
        service: backend
  template:
    metadata:
      labels:
        app: nginx
        service: backend
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: default
spec:
  selector:
    service: backend
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
```

```text
deployment.apps/nginx-deployment created
service/nginx-svc created
```

### Apache

```shell
kubectl create -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
  namespace: default
  labels:
    service: backend
    app: apache
spec:
  replicas: 1
  selector:
    matchLabels:
        service: backend
        app: apache
  template:
    metadata:
      labels:
        service: backend
        app: apache
    spec:
      containers:
      - name: apache
        image: httpd
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: apache-svc
  namespace: default
spec:
  selector:
    service: backend
    app: apache
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
```

```text
deployment.apps/apache-deployment created
service/apache-svc created
```

### Create an Istio Virtual service

```shell
kubectl create -f - << EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: simple-vs
  namespace: default
spec:
  gateways:
  - istio-ingress/default
  hosts:
  - '*'
  http:
  - route:
    - destination:
        host: balanced-destinatio-route
EOF
```

### Create a Destination Route



```
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```


```text
virtualservice.networking.istio.io/reviews created
```