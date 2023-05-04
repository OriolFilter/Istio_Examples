---
gitea: none
include_toc: true
---

# Description

The [previous example](../../01-Getting_Started/03-hello_world_1_service_2_deployments_managed_version), we use the values set in the headers from the `HTTP` request received to route the traffic to different backends.

Additionally, we configure a default rule where the unmatched traffic will go.

This example configures:

    Generic Kubernetes resources:
    - 1 Namespace
    - 1 Service
    - 3 Deployments
    
    Istio resources:
    - 1 Gateway
    - 1 Virtual Service
    - 1 Destination rule

# Based on

- [03-hello_world_1_service_2_deployments_managed_version](../../01-Getting_Started/03-hello_world_1_service_2_deployments_managed_version)

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

We will create 3 different deployments where the traffic will be distributed, targeting the labels set to the pods.

Deployments created:

- helloworld-v0
- helloworld-v1
- helloworld-v2

### helloworld-v0

Deploys a Whoami server that listens for the port `80`.

On this deployment, we attributed the label `version` set to `v0`, this will be used by the [DestinationRule](#destinationrule) resource to target this deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v0
  labels:
    app: helloworld
    version: v0
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      version: v0
  template:
    metadata:
      labels:
        app: helloworld
        version: v0
    spec:
      containers:
        - name: helloworld
          image: containous/whoami
          resources:
            requests:
              cpu: "100m"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
```

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

The configuration set, targets the [gateway created](#gateway) as well of not limiting the traffic to any specific host.

We will create 3 rules that target `HTTP` (including `HTTPS` and `HTTP2` traffic), which are declared with the following names:

- firefox
- curl
- default

The `firefox` rule is intended to receive traffic coming from a Firefox web browser.

The `curl` rule is intended to receive traffic coming from the `curl` command line package/command.  

The `default` rule is intended to receive traffic not matched by the 2 rules of above (as long meets certain criteria).

All the rules created will match on the following topics:

- The path directory will be `/helloworld`.

- The destination service will be `helloworld.default.svc.cluster.local` to the port `80`.

- Rewrite URL path to `/` to not create conflicts with the backend.

The rules will differ on the following topics:

- Each rule will target a specific content from the header `user-agent`, except of default.

- Each rule will target a different `subset` to distribute the traffic between the [Deployment resources created](#deployment).

> **Note**\
> If you usage not familiar with the usage of `subsets`, I strongly recommend to check the following example:
> - [03-hello_world_1_service_2_deployments_managed_version](../../01-Getting_Started/03-hello_world_1_service_2_deployments_managed_version) 

> **Note**\
> The declaration order from the resources is important, as the elements created further above on the list will have priority over the rules set under them.\
> This would mean that if we declared the `default` matching rule first, it would overtake the traffic meant to reach the rules `firefox` and `curl`.
> For more information regarding this matter, refer to the following Official Istio documentation regarding [the HttpMatchRequest field from Istio VirtualServices](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMatchRequest).


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
    - name: firefox
      match:
        - uri:
            exact: /helloworld
          headers:
            user-agent:
              regex: '.*Firefox.*'
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
            subset: nginx
      rewrite:
        uri: "/"
    - name: curl
      match:
        - headers:
            user-agent:
              regex: '.*curl.*'
          uri:
            exact: /helloworld
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
            subset: apache
      rewrite:
        uri: "/"
    - name: default
      match:
        - uri:
            exact: /helloworld
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
            subset: default
      rewrite:
        uri: "/"
```

## DestinationRule

This `DestinationRule` interferes with the traffic with destination `helloworld.default.svc.cluster.local`.

Contains 3 subsets defined, where each one will target a different backend.


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld.default.svc.cluster.local # Destination that will "interject"
spec:
  host: helloworld.default.svc.cluster.local # Full destination service, lil better for consistency
  subsets:
    - name: default
      labels:
        version: v0
    - name: nginx
      labels:
        version: v1
    - name: apache
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
deployment.apps/helloworld-v0 created
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
watch -n 2 kubectl get deployment helloworld-v{0..2} 
```
```text 
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-v0   1/1     1            1           23s
helloworld-v1   1/1     1            1           23s
helloworld-v2   1/1     1            1           23s
```

## Test the service

Now it's time to test the rules created in the [VirtualService configuration](#virtualservice), for such reminder that:

- Curl should return traffic from the Apache deployment.
- Firefox should return traffic from the Nginx deployment.
- The rest of the traffic should return us a response from the Whoami deployment. (as long it matches the rest of criteria set in the rule).

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

Using the command `curl`, results in receiving content from the Apache deployment.

```text
curl 192.168.1.50/helloworld -s                     
```
```text
<html><body><h1>It works!</h1></body></html>
```

### Firefox browser

Through using the Firefox web browser, the page accessed is the Nginx deployment. 

![firefox.png](src%2Ffirefox.png)

### Default
To trigger the "default" rule, you can be quite creative there, like using another browser some other tool.

In my case I will be using the package `wget` as it is more simple for myself.


```shell
wget 192.168.1.50/helloworld -O /dev/stdout -q
```

```text
Hostname: helloworld-v0-64fc7d6ccb-vdljp
IP: 127.0.0.1
IP: ::1
IP: 172.17.247.19
IP: fe80::dc11:59ff:fe15:f6c3
RemoteAddr: 127.0.0.6:51179
GET / HTTP/1.1
Host: 192.168.1.50
User-Agent: Wget/1.21.3
Accept: */*
Accept-Encoding: identity
X-B3-Parentspanid: 7a17876e5182b4a1
X-B3-Sampled: 0
X-B3-Spanid: 0aa085397844c696
X-B3-Traceid: 3c9474b384a9f33c7a17876e5182b4a1
X-Envoy-Attempt-Count: 1
X-Envoy-Internal: true
X-Envoy-Original-Path: /helloworld
X-Forwarded-Client-Cert: By=spiffe://cluster.local/ns/default/sa/default;Hash=bf05b3ab5654afeecc50ea6b136708517bcfdec088978a2a675759d52aa207aa;Subject="";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account
X-Forwarded-For: 192.168.1.10
X-Forwarded-Proto: http
X-Request-Id: 46d7dbe8-ba46-4703-b1a3-cdecf2f93d1e
```


## Cleanup

Finally, a cleanup from the resources deployed.

```shell
kubectl delete -f ./
```
```text
deployment.apps "helloworld-v0" deleted
deployment.apps "helloworld-v1" deleted
deployment.apps "helloworld-v2" deleted
destinationrule.networking.istio.io "helloworld.default.svc.cluster.local" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
service "helloworld" deleted
```



# Links of interest

- https://istio.io/latest/docs/reference/config/networking/virtual-service/

- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMatchRequest

- https://istio.io/latest/docs/reference/config/networking/destination-rule/#