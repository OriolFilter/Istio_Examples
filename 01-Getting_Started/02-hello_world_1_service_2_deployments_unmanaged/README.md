---
gitea: none
include_toc: true
---

# Description

This example deploys the same infrastructure as the  [previous example](../01-hello_world_1_service_1_deployment), this time containing 2 deployments under the same service.

This example configures:

    Generic Kubernetes resources:
    - 1 Service
    - 2 Deployments
    
    Istio resources:
    - 1 Gateway
    - 1 Virtual Service


# Based on

- [01-hello_world_1_service_1_deployment](../01-hello_world_1_service_1_deployment)

# Configuration

## Service

Creates a service named `helloworld`.

This service listens for the port `80` expecting `HTTP` traffic and will forward the incoming traffic towards the port `80` from the destination pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
    service: helloworld
spec:
  ports:
    - port: 80
      name: http
  selector:
    app: helloworld
```

## Deployment

### helloworld-v1

Deploys a Nginx server that listens for the port `80`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v1
  labels:
    app: helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          image: nginx
          resources:
            requests:
              cpu: "100m"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
```

### helloworld-v2

Deploys an Apache server that listens for the port `80`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v2
  labels:
    app: helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          image: httpd
          resources:
            requests:
              cpu: "100m"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
```

## Gateway

Deploys an Istio gateway that's listening to the port `80` for `HTTP` traffic.

It doesn't filter for any specific host.

The `selector` field is used to "choose" which Istio Load Balancers will have this gateway assigned to.

The Istio `default` profile creates a Load Balancer in the namespace `istio-system` that has the label `istio: ingressgateway` set, allowing us to target that specific Load Balancer and assign this gateway resource to it.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: helloworld-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

## VirtualService

The Virtual Service resources are used to route and filter the received traffic from the gateway resources, and route it towards the desired destination.

On this example we select the gateway `helloworld-gateway`, which is the [gateway that 's described in the `Gateway` section](#gateway).

On this resource, we are also not limiting the incoming traffic to any specific host, allowing for all the incoming traffic to go through the rules set.

Here we created a rule that will be applied on `HTTP` related traffic (including `HTTPS` and `HTTP2`) when the destination path is exactly `/helloworld`.

This traffic will be forwarded to the port `80` of the destination service `helloworld` (the full path URL equivalent would be `helloworld.$NAMESPACE.svc.cluster.local`).

Additionally, there will be an internal URL rewrite set, as if the URL is not modified, it would attempt to reach to the `/helloworld` path from the Nginx deployment, which currently has no content and would result in an error code `404` (Not found).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-vs
spec:
  hosts:
    - "*"
  gateways:
    - helloworld-gateway
  http:
    - match:
        - uri:
            exact: /helloworld
      route:
        - destination:
            host: helloworld
            port:
              number: 80
      rewrite:
        uri: "/"
```


# Walkthrough

## Deploy resources

Deploy the resources.

```shell
kubectl apply -f ./
```
```text 
deployment.apps/helloworld-v1 created
deployment.apps/helloworld-v2 created
gateway.networking.istio.io/helloworld-gateway created
service/helloworld created
virtualservice.networking.istio.io/helloworld-vs created
```

## Wait for the pods to be ready

Wait for the Apache and Nginx deployments to be up and ready.

```shell
kubectl get deployment helloworld-v{1..2} -w
```
```text 
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-v1   1/1     1            1           4m1s
helloworld-v2   1/1     1            1           4m1s
```

## Test the service

### Get LB IP

To perform the desired tests, we will need to obtain the IP Istio Load Balancer that we selected in the [Gateway section](#gateway).

On my environment, the IP is the `192.168.1.50`.

```shell
kubectl get svc -l istio=ingressgateway -A
```
```text
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```

### Curl

Performing a series of `curl` requests, we can observe how the response iterate between the Nginx and Apache backends. 

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<h1>Welcome to nginx!</h1>
```

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<html><body><h1>It works!</h1></body></html>
```

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<h1>Welcome to nginx!</h1>
```

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<html><body><h1>It works!</h1></body></html>
```

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<h1>Welcome to nginx!</h1>
```

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<html><body><h1>It works!</h1></body></html>
```

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<h1>Welcome to nginx!</h1>
```

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<h1>Welcome to nginx!</h1>
```

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
```
```text
<html><body><h1>It works!</h1></body></html>
```

## Cleanup

Finally, a cleanup from the resources deployed.

```shell
kubectl delete -f ./
```
```text
deployment.apps "helloworld-v1" deleted
deployment.apps "helloworld-v2" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
service "helloworld" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
```