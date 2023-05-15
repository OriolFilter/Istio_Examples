---
gitea: none
include_toc: true
---

# Description

This example deploys the same infrastructure as the [previous example](../../01-Getting_Started/01-hello_world_1_service_1_deployment), and restrict which `VirtualService` Istio resources can access/select the `Gateway` Istio resource, based on the `VirtualService` namespace.

The domain host targeted will be `my.domain`.

This example configures:

    Generic Kubernetes resources:
    - 1 Service
    - 1 Deployment
    - 1 Namespace
    
    Istio resources (`default` namespace):
    - 1 Gateway
    -  Virtual Service

    Istio resources (`foo`namespace):
    - 1 Virtual Service

> **Note:**\
> I don't intend to explain thing related to Kubernetes unless necessary.

# Based on

- [01-hello_world_1_service_1_deployment](../../01-Getting_Started/01-hello_world_1_service_1_deployment

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

## Namespace

Creates a namespace named `foo`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: foo
```

## Gateway

Deploys an Istio gateway that's listening to the port `80` for `HTTP` traffic.

The gateway won't target any specific host domain, yet limits the `VirtualService` Istio resources that can target this gateway, limiting its access to the `VirtualServices` Istio resources created in the `foo` namespace.

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
        name: http-b
        protocol: HTTP
      hosts:
        - "foo/*"
```

## VirtualService

We will create two `VirtualServices` with the same configuration, only difference will be the namespace they are created onto (and the destination path), this will be used to test if the [`Gateway` namespace restriction configured](#gateway) is being applied to the `VirtualService` resources as desired. 

On this example we select the gateway `helloworld-gateway`, which is the [gateway that 's described in the `Gateway` section](#gateway).

On this resource, we are also not limiting the incoming traffic to any specific host, allowing for all the incoming traffic to go through the rules set.

Additionally, there will be an internal URL rewrite set, as if the URL is not modified, it would attempt to reach to the `/helloworld` path from the Nginx deployment, which currently has no content and would result in an error code `404` (Not found).


## helloworld-foo

`VirtualService` created in the namespace `foo`.

Here we created a rule that will be applied on `HTTP` related traffic (including `HTTPS` and `HTTP2`) when the destination path is exactly `/helloworld`.

This traffic will be forwarded to the port `80` of the destination service `helloworld.default.svc.cluster.local`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-foo
  namespace: foo
spec:
  hosts:
    - "*"
  gateways:
    - default/helloworld-gateway
  http:
    - match:
        - uri:
            exact: /helloworld
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
      rewrite:
        uri: "/"
```

## helloworld-default

`VirtualService` created in the namespace `default`.

Here we created a rule that will be applied on `HTTP` related traffic (including `HTTPS` and `HTTP2`) when the destination path is exactly `/failure`.

This traffic will be forwarded to the port `80` of the destination service `helloworld.default.svc.cluster.local`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-default
  namespace: default
spec:
  hosts:
    - "*"
  gateways:
    - default/helloworld-gateway
  http:
    - match:
        - uri:
            exact: /failure
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
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
namespace/foo created
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
service/helloworld created
virtualservice.networking.istio.io/helloworld-foo created
virtualservice.networking.istio.io/helloworld-default created
```

## Wait for the deployment to be ready

Wait for the Nginx deployment to be up and ready.

```shell
kubectl get deployment helloworld-nginx -w 
```
```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-nginx   1/1     1            1           44s
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

### Curl /helloworld

When performing a curl towards the destination path, as we are not using the domain host specified in the [gateway resource](#gateway), we are failing to match any rule.

```shell
 curl 192.168.1.50/helloworld -I
```
```text
HTTP/1.1 404 Not Found
date: Wed, 10 May 2023 08:25:26 GMT
server: istio-envoy
transfer-encoding: chunked
```

### Curl my.domain/helloworld

We can "fake" the destination domain by modifying the `Host` header.

After setting that up, and attempting to curl the destination, we receive a positive response from the Nginx backend. 

```shell
curl 192.168.1.50/helloworld -s -HHOST:my.domain | grep "<title>.*</title>"
```
```text
<title>Welcome to nginx!</title>
```


## Cleanup

Finally, a cleanup from the resources deployed.

```shell
kubectl delete -f ./
```
```text
namespace "foo" deleted
deployment.apps "helloworld-nginx" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
service "helloworld" deleted
virtualservice.networking.istio.io "helloworld-foo" deleted
virtualservice.networking.istio.io "helloworld-default" deleted
```

# Links of interest

- https://istio.io/latest/docs/reference/config/networking/gateway/