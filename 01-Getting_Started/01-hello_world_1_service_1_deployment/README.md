---
gitea: none
include_toc: true
---


# Description 

This is the most basic example, most of the examples spread through this [repository](../../) will be using variants of this. 

This example configures:

    Generic Kubernetes resources:
    - 1 Service
    - 1 Deployment
    
    Istio resources:
    - 1 Gateway
    - 1 Virtual Service

> **Note:**\
> I don't intend to explain thing related to Kubernetes unless necessary.

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
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
service/helloworld created
virtualservice.networking.istio.io/helloworld-vs created
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

Due to accessing the path `/helloworld`, we are triggering the rule set on the [VirtualService configuration](#virtualservice), sending a request to the Nginx backend and returning us its contents. 

```shell
curl 192.168.1.50/helloworld -s | grep "<title>.*</title>" 
```
```text
<title>Welcome to nginx!</title>
```

### Curl /other

What happens if we access a path or URL that doesn't trigger any rule?


```shell
curl 192.168.1.50/other -s -I 
```
```text
HTTP/1.1 404 Not Found
date: Sun, 30 Apr 2023 22:16:30 GMT
server: istio-envoy
transfer-encoding: chunked
```

We receive a status code `404`.

I would like to put emphasis on the following line returned:

```text
server: istio-envoy
```

This means that the contents returned was performed by the Istio service, therefore, the request was able to reach Istio and received a response from it.

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