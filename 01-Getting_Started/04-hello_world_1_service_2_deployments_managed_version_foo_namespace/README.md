---
gitea: none
include_toc: true
---

# Description

Based on the [previous example](../03-hello_world_1_service_2_deployments_managed_version), we create the previous deployments in a different namespace than the Istio VirtualService object, in this case we create them in the namespace `foo`.

This is example is based on the following post regarding [canary deployments on Istio](https://istio.io/latest/blog/2017/0.1-canary/).

This example configures:

    Generic Kubernetes resources:
    - 1 Namespace
    - 1 Service
    - 2 Deployments
    
    Istio resources:
    - 1 Gateway
    - 1 Virtual Service
    - 1 Destination rule

Additionally, for consistency, now the resources are being created in the `default` namespace.

On this example, the `service`, the `deployment`, and the Istio `DestinationRule` are being created in the namespace `foo`.

As well, on the [VirtualService section](#virtualservice), we are targeting the service `helloworld` by the full URL, where on previous examples it was targeted by `helloworld`, now it's targeted by `helloworld.default.svc.cluster.local`.

# Based on

- [03-hello_world_1_service_2_deployments_managed_version](../03-hello_world_1_service_2_deployments_managed_version)

# Configuration

## Namespace

Creates a namespace named `foo`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: foo
```

## Service

Creates a service named `helloworld`.

This service listens for the port `80` expecting `HTTP` traffic and will forward the incoming traffic towards the port `80` from the destination pod.

The service is created in the namespace `foo`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  namespace: foo
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

On this deployment, we attributed the label `version` set to `v1`, this will be used by the [DestinationRule](#destinationrule) resource to target this deployment.

The deployment is created in the namespace `foo`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v1
  namespace: foo
  labels:
    app: helloworld
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      version: v1
  template:
    metadata:
      labels:
        app: helloworld
        version: v1
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

On this deployment, we attributed the label `version` set to `v2`, this will be used by the [DestinationRule](#destinationrule) resource to target this deployment.

The deployment is created in the namespace `foo`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v2
  namespace: foo
  labels:
    app: helloworld
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      version: v2
  template:
    metadata:
      labels:
        app: helloworld
        version: v2
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
  namespace: default
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

The Virtual Service resources are used to route and filter the received traffic from the gateway resources, and route it towards the desired destination.

On this example we select the gateway `helloworld-gateway`, which is the [gateway that 's described in the `Gateway` section](#gateway).

On this resource, we are also not limiting the incoming traffic to any specific host, allowing for all the incoming traffic to go through the rules set.

Here we created a rule that will be applied on `HTTP` related traffic (including `HTTPS` and `HTTP2`) when the destination path is exactly `/helloworld`.

This traffic will be forwarded to the port `80` of the destination service `helloworld` located in the namespace `foo`.

There will be an internal URL rewrite set, as if the URL is not modified, it would attempt to reach to the `/helloworld` path from the Nginx deployment, which currently has no content and would result in an error code `404` (Not found).

Also, there's been configured 2 destinations under the same rule, each one with a `subset` set, which will be used by the [DestinationRule](#destinationrule) object to manage the traffic from each destination.

As well, where each one of the destinations mentioned, has a `weight` set, this value will be used to distribute the incoming requests towards the specified subsets.

> **Note:**
> A 20% of the traffic will be sent to the `subset` v1, meanwhile 80% will be sent to the `subset` v2.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-vs
  namespace: default
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
            host: helloworld.foo.svc.cluster.local
            port:
              number: 80
            subset: v1
          weight: 20
        - destination:
            host: helloworld.foo.svc.cluster.local
            port:
              number: 80
            subset: v2
          weight: 80
      rewrite:
        uri: "/"
```
## DestinationRule

This `DestinationRule` interferes with the traffic with destination `helloworld.foo.svc.cluster.local`.

Contains 2 subsets defined, where each one will target a different backend.

A reminder that the `version: v1` was given to the Nginx backend, meanwhile the Apache backend had set `version: v2`.

This resource is created in the namespace `foo`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld
  namespace: foo
spec:
  host: helloworld.foo.svc.cluster.local # Full destination service, lil better for consistency
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```


## Deploy resources

Deploy the resources.

```shell
kubectl apply -f ./
```
```text 
deployment.apps/helloworld-v1 created
deployment.apps/helloworld-v2 created
destinationrule.networking.istio.io/helloworld.default.svc.cluster.local created
gateway.networking.istio.io/helloworld-gateway created
service/helloworld created
virtualservice.networking.istio.io/helloworld-vs created
```

## Wait for the pods to be ready

Wait for the Apache and Nginx deployments to be up and ready.

```shell
watch -n 2 kubectl get deployment helloworld-v{1..2} 
```
```text 
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-v1   1/1     1            1           58s
helloworld-v2   1/1     1            1           58s
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

As we can see, we can reach out to the service, now located in the namespace `foo`.

As well, if we check the ration on which we reached the backends, we can still see how the [VirtualService](#virtualservice) weight configured to the subsets is still being applied.

> Nginx: 3\
> Apache: 9

```text
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<h1>Welcome to nginx!</h1>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<h1>Welcome to nginx!</h1>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<h1>Welcome to nginx!</h1>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>

$ 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>
```


## Cleanup

Finally, a cleanup from the resources deployed.

```shell
kubectl delete -f ./
```
```text
namespace "foo" deleted
deployment.apps "helloworld-v1" deleted
deployment.apps "helloworld-v2" deleted
destinationrule.networking.istio.io "helloworld" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
service "helloworld" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
```