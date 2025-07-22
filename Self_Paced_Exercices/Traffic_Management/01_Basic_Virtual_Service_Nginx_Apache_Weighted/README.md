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
  replicas: 3
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
  replicas: 3
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
        host: nginx-svc
      weight: 50
    - destination:
        host: apache-svc
      weight: 50
EOF
```

```text
virtualservice.networking.istio.io/reviews created
```

## Test the virtual service

```shell
kubectl get svc -owide -n istio-ingress 
```

```text
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE   SELECTOR
istio-ingress   LoadBalancer   10.96.22.181   172.18.10.10   15021:31738/TCP,80:30305/TCP,443:30847/TCP   16m   app=istio-ingress,istio=ingress
```


```shell
curl 172.18.10.10
```

Similarly to the 

```text
<html><body><h1>It works!</h1></body></html>
```

## Update for one-sided weight

```shell
kubectl apply -f - << EOF
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
        host: nginx-svc
      weight: 9
    - destination:
        host: apache-svc
      weight: 1
EOF
```


### Test changes

```shell
curl 172.18.10.10
```

Unlike before where the traffic was fairly distributed, here the constant is to get the Nginx response, except the periodical Apache occurrence.

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```