---
gitea: none
include_toc: true
---

# Description

Based on the [previous example](../../01-Getting_Started/01-hello_world_1_service_1_deployment), we configure the [VirtualService](#virtualservice) to return a specific response to the client with the contents of our choice.

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

We configure 2 rules for HTTP traffic (this includes `HTTPS` and `HTTP2`, this will be my last warning about this).

The first rule configured will match when the requested path is `/helloworld`.

This traffic will be forwarded to the service `helloworld.default.svc.cluster.local` with port `80`.

The second rule, which will manage the rest of the traffic, will return a response with status code `404`, and body content set to `Page Not Found`.

Additionally, for consistency and good practices purposes, we also specified the content type of the page to `text/plain`. 

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: helloworld-vs
spec:
  hosts:
    - "*"
  gateways:
    - helloworld-gateway
  http:
    - name: helloworld
      match:
        - uri:
            exact: /helloworld
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
      rewrite:
        uri: "/"
    - name: default
      directResponse:
        status: 404
        body:
          string: "Page Not Found"
      headers:
        response:
          set:
            content-type: "text/plain"
```

# Walkthrough

## Deploy resources

Deploy the resources.

```shell
kubectl apply -f ./
```
```text 
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
service/helloworld created
virtualservice.networking.istio.io/helloworld-vs created
```

## Wait for the pods to be ready

Wait for the Apache and Nginx deployments to be up and ready.

```shell
kubectl get deployment helloworld-nginx -w
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

### helloworld

As expected, we receive the contents from the Nginx contents

```shell
curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
```
```text
<h1>Welcome to nginx!</h1>
```
### other paths

#### /

We receive the contents "Page Not Found".

```shell
curl 192.168.1.50/   
```

```text
Page Not Found 
```

Checking the headers, we also can see the status code is set to `404`, as well of the content-type we specified on the [VirtualService configuration](#virtualservice).

```shell
curl 192.168.1.50/ -I   
```

```text
HTTP/1.1 404 Not Found
content-type: text/plain
content-length: 14
date: Thu, 04 May 2023 01:18:50 GMT
server: istio-envoy
```



#### /found

We can confirm that this behavior applies to other paths (as it is matching the "default" rule we set in [VirtualService configuration]).

```shell
curl 192.168.1.50/found
```

```text
Page Not Found
```

```shell
curl 192.168.1.50/found -I   
```

```text
HTTP/1.1 404 Not Found
content-type: text/plain
content-length: 14
date: Thu, 04 May 2023 01:20:40 GMT
server: istio-envoy
```


## Cleanup

Finally, a cleanup from the resources deployed.

```shell
kubectl delete -f ./
```
```text
deployment.apps "helloworld-nginx" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
service "helloworld" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
```

# Links of interest

- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPDirectResponse

- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPBody
