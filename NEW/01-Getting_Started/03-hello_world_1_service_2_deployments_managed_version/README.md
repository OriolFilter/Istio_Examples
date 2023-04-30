---
gitea: none
include_toc: true
---

# Description

Based on the [previous example](../02-hello_world_1_service_2_deployments_unmanaged), where we created 2 deployments under the same service, we will attribute different tags to each Deployment, and afterwards, through the usage of a [DestinationRule](#destinationrule), internationally target the desired backend to route the traffic towards it. 

This is example is based on the following post regarding [canary deployments on Istio](https://istio.io/latest/blog/2017/0.1-canary/).

This example configures:

    Generic Kubernetes resources:
    - 1 Service
    - 2 Deployments
    
    Istio resources:
    - 1 Gateway
    - 1 Virtual Service
    - 1 Destination rule

Additionally, for consistency, now the resources are being created in the `default` namespace.

As well, on the [VirtualService section](#virtualservice), we are targeting the service `helloworld` by the full URL, where on previous examples it was targeted by `helloworld`, now it's targeted by `helloworld.default.svc.cluster.local`.

# Based on

- [02-hello_world_1_service_2_deployments_unmanaged](../02-hello_world_1_service_2_deployments_unmanaged)

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

On this deployment, we attributed the label `version` set to `v1`, this will be used by the [DestinationRule](#destinationrule) resource to target this deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v1
  labels:
    app: helloworld
    version: v1
  namespace: default
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

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v2
  labels:
    app: helloworld
    version: v2
  namespace: default
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

This traffic will be forwarded to the port `80` of the destination service `helloworld` (the full path URL equivalent would be `helloworld.$NAMESPACE.svc.cluster.local`).

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
            host: helloworld.default.svc.cluster.local
            #            host: helloworld (OLD)
            port:
              number: 80
            subset: v1
          weight: 20
        - destination:
            #            host: helloworld (OLD)
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
            subset: v2
          weight: 80
      rewrite:
        uri: "/"
```

## Destination rule

This `DestinationRule` interferes with the traffic with destination `helloworld.default.svc.cluster.local`.

Contains 2 subsets defined, where each one will target a different backend.

A reminder that the `version: v1` was given to the Nginx backend, meanwhile the Apache backend had set `version: v2`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  # name: helloworld (OLD)
  name: helloworld.default.svc.cluster.local # Destination that will "interject"
  namespace: default
spec:
  #  host: helloworld # destination service (OLD)
  host: helloworld.default.svc.cluster.local # Full destination service, lil better for consistency
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
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

By performing a series of curls, we can notice how the output received iterates between Nginx and Apache.

If we take into account the configuration set, and we review the results, we can notice how the ratio is close to the one configured in the [VirtualService](#virtualservice) section.

> Nginx instances (v1): 2 \
> Apache instances (v2): 9

```text
$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<h1>Welcome to nginx!</h1>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<h1>Welcome to nginx!</h1>
```

## Check Istio internal configurations created

Using the command `istioctl x describe pod $POD`, we can see which Istio configuration is currently attributed to that specific pod.

### v1

We can notice the following line:

`Weight 20%`

Which matches the configuration set in the [VirtualService](#virtualservice) configuration.

```sh
istioctl x describe pod $(kubectl get pod -l app=helloworld,version=v1 -o jsonpath='{.items[0].metadata.name}')
```
```text
Pod: helloworld-v1-7454b56b86-4cksf
   Pod Revision: default
   Pod Ports: 80 (helloworld), 15090 (istio-proxy)
--------------------
Service: helloworld
   Port: http 80/HTTP targets pod port 80
DestinationRule: helloworld for "helloworld.default.svc.cluster.local"
   Matching subsets: v1
      (Non-matching subsets v2)
   No Traffic Policy
--------------------
Effective PeerAuthentication:
   Workload mTLS mode: PERMISSIVE


Exposed on Ingress Gateway http://192.168.1.50
VirtualService: helloworld-vs
   Weight 20%
   /helloworld
```

### v2

We can notice the following line:

`Weight 80%`

Which matches the configuration set in the [VirtualService](#virtualservice) configuration.

```shell
istioctl x describe pod `kubectl get pod -l app=helloworld,version=v2 -o jsonpath='{.items[0].metadata.name 
```
```text
Pod: helloworld-v2-64b5656d99-5bwgr
   Pod Revision: default
   Pod Ports: 80 (helloworld), 15090 (istio-proxy)
--------------------
Service: helloworld
   Port: http 80/HTTP targets pod port 80
DestinationRule: helloworld for "helloworld.default.svc.cluster.local"
   Matching subsets: v2
      (Non-matching subsets v1)
   No Traffic Policy
--------------------
Effective PeerAuthentication:
   Workload mTLS mode: PERMISSIVE


Exposed on Ingress Gateway http://192.168.1.50
VirtualService: helloworld-vs
   Weight 80%
   /helloworld
```

## Cleanup

Finally, a cleanup from the resources deployed.

```shell
kubectl delete -f ./
```
```text
deployment.apps "helloworld-v1" deleted
deployment.apps "helloworld-v2" deleted
destinationrule.networking.istio.io "helloworld.default.svc.cluster.local" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
service "helloworld" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
```


# Links of Interest

- https://istio.io/latest/blog/2017/0.1-canary/
- https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule