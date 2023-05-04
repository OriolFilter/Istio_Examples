---
gitea: none
include_toc: true
---

# Description

Based on the [previous example](../../01-Getting_Started/01-hello_world_1_service_1_deployment), we configure the [VirtualService](#virtualservice) to internally rewrite the destination URL.

This is useful, as if for example we have a rule that targets the traffic with destination path `/helloworld`, when we connect to the backend, the path that the request contains will also be `/helloworld`, and unless the destination service is already build around this and/or is ready to manage traffic with such destination, we will receive a status code 404 meaning that the page destination was not found.

If we internally rewrite such traffic to the root directory (`/`), we can interact with the root path from the destination service without issues, without the need of specifically altering the behavior of the destination service due this architectural requirement.

Additionally, we also configure a second rule that won't have the URL path rewrite configured, as it will allow us to compare the behaviors.  

This example configures:

    Generic Kubernetes resources:
    - 1 Service
    - 1 Deployments
    
    Istio resources:
    - 1 Gateway
    - 1 Virtual Service


# Based on

- [01-hello_world_1_service_1_deployment](../../01-Getting_Started/01-hello_world_1_service_1_deployment)

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

Deploys a Nginx server that listens for the port `80`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-nginx
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
          imagePullPolicy: IfNotPresent #Always
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
    istio: ingressgateway # use istio default controller
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

## VirtualService

The configuration set, targets the [gateway created](#gateway) as well of not limiting the traffic to any specific host.

We configure 2 HTTP rules.

The first rule will match when the requested path is `/helloworld`.

Internally, we will rewrite the URL path, from `/helloworld` to `/`, as otherwise it will result in status code 404 due not containing such destination in the service, since we are using the default Nginx image.

The second rule will math with the path `/norewrite`, and won't have the rewrite URL path setting configured. This rule will be used to compare behaviors.


Both rules will connect with the backend service `helloworld.default.svc.cluster.local` with port `80`.

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
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
    - match:
        - uri:
            exact: /norewrite
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
```

# Walkthrough

## Deploy resources

Deploy the resources.

```shell
kubectl apply -f ./
```
```text 
deployment.apps/helloworld-nginx created
service/helloworld created
virtualservice.networking.istio.io/helloworld-vs created
gateway.networking.istio.io/helloworld-gateway created
```

## Wait for the pods to be ready

Wait for the Nginx deployment to be up and ready.

```shell
kubectl get deployment helloworld-nginx -w
```
```text 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-nginx   1/1     1            1           2m47s
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

### helloworld

Due to rewriting the URL path internally, we are able to connect to the backend root path (`/`) 

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<h1>Welcome to nginx!</h1>
```

### norewrite

As expected, due the backend service not having a destination path named `/norewrite`, we receive a status code 404 as well of their pertinent service error page.

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<center><h1>404 Not Found</h1></center>
```

## Cleanup

Finally, a cleanup from the resources deployed.

```shell
kubectl delete -f ./
```
```text
deployment.apps "helloworld-nginx" deleted
service "helloworld" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
```

# Links of interest

- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPDirectResponse

- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPBody
















# There were no changes respective to that version

Through rewriting the URI we can point to the root directory from nginx.

```yaml
      rewrite:
        uri: "/"
```





## The idea is that this rewrite is handled "internally" by Istio, not by the Client that started the request


## Practical usages:



If we refactor our application, and for example we previously where hosting an API to the URL `/apiV1` and now it's being hosted in `/api/V1`, we can do the following rule:


```yaml
    - match:
        - uri:
            exact: /apiV1
      route:
        - destination:
            host: mynewapi # the service destination/target
            port:
              number: 80 # whatever port it is
      rewrite:
        uri: "/api/V1"
```

Or if we "upgraded" the API, and the new API (v2) is retro-compatible with the old API (v1), we could do the following to force all the usages from the old API to be handled by the newer version:

```yaml
    - match:
        - uri:
            exact: /api/V1
      route:
        - destination:
            host: mynewapi # the service destination/target
            port:
              number: 80 # whatever port it is
      rewrite:
        uri: "/api/V2"
```
