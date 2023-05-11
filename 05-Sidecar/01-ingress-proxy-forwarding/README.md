---
gitea: none
include_toc: true
---

# Description

This example deploys the same infrastructure as the [previous example](../../01-Getting_Started/01-hello_world_1_service_1_deployment), configures the **sidecar** `envoy-proxy`/`istio-proxy`/`sidecar-proxy` on the pods created, to forward the traffic incoming from the port `8080` to the port `80`.

This example configures:

    Generic Kubernetes resources:
    - 1 Service
    - 1 Deployment
    
    Istio resources:
    - 1 Gateway
    - 1 Virtual Service
    - 1 Sidecar configration

# Based on

- [01-hello_world_1_service_1_deployment](../../01-Getting_Started/01-hello_world_1_service_1_deployment)

# Configuration

`etc -> "POD" -> sidecar -> service container`

## Service

Creates a service named `helloworld`.

This service listens for the port `8080` expecting `HTTP` traffic and will forward the incoming traffic towards the port `8080` from the destination pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
spec:
  ports:
    - port: 8080
      name: http
  selector:
    app: helloworld
```

## Deployment

Deploys a Nginx server that listens for the port `80`.

We can notice how in the service we opened the port `8080` and in the deployment we are listening to the port `80`, more about this in the [Sidecar Section](#sidecar).

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

Here we created a rule that will be applied on `HTTP` related traffic when the destination path is exactly `/helloworld`.

This traffic will be forwarded to the port `8080` of the destination service `helloworld` (the full path URL equivalent would be `helloworld.$NAMESPACE.svc.cluster.local`).

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

## Sidecar

This will configure the sidecar configuration from the `envoy-proxy` in each pod.

`workloadSelector` will be used to select the target pods, where, on this scenario, it will target the pods that have the label set `app: helloworld`.

The ingress configuration set, will listen for the port `8080` from the pod, and forward it to the pod's port `80` through the loopback (127.0.0.1) IP.

On this scenario we are performing a simple `8080` to `80` redirect.

> **Note:**\
> A reminder that a `POD` is an object that groups container(s).

+ more notes:

- workloadSelector:

> `workloadSelector` is used to target the `PODS`, on which apply this sidecar configuration. \
> Bear in mind that this configuration doesn't target kinds `Service`, nor `Deployment`, it's applied to a kind `Pod` or `ServiceEntry` \
> If there is no `workloadSelector` specified, it will be used as default configuration for the namespace on which was created. \
> More info in the [Istio documentation for workloadSelector](https://istio.io/latest/docs/reference/config/networking/sidecar/#WorkloadSelector)

- ingress:

> Configure the behavior of the ingress traffic.\
> On this "grabs"/targets the ingress traffic with port 8080, and forwards it to the port IP `127.0.0.1` (loopback) respective to the destination pod, with the destination port set to 80, which is the port that the service is currently listening to.


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: helloworld-sidecar
spec:
  workloadSelector:
    labels:
      app: helloworld
  ingress:
    - port:
        number: 8080
        protocol: HTTP
        name: ingressport
      defaultEndpoint: 127.0.0.1:80
```

# Run example

## Deploy resources

```shell
kubectl apply -f ./
```

```text
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
service/helloworld created
sidecar.networking.istio.io/helloworld-sidecar created
virtualservice.networking.istio.io/helloworld-vs created
```

## Wait for the pods to be ready

```shell
kubectl get deployment helloworld-nginx -w
```

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-nginx   1/1     1            1           39s
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

We can perform a curl towards the destination.

A reminder that the configuration set in the [service](#service) created, it's listening to the port `8080` and forwarding the traffic to the same pod (`8080`).

As well on the Istio's [VirtualService](#virtualservice), we configured the destination port as `8080`.

Yet, on the [Sidecar](#sidecar) configuration, we are redirecting the ingress traffic from the port `8080`, to the port `80`.

```shell
curl 192.168.1.50/helloworld -s | grep "<title>.*</title>"
```
```text
<title>Welcome to nginx!</title>
```

### Delete the sidecar configuration to force failure.

As per the moment let's delete the `sidecar` configuration deployed.

```shell
kubectl delete sidecars.networking.istio.io helloworld-sidecar
```
```text
sidecar.networking.istio.io "helloworld-sidecar" deleted
```

### Curl again

After deleting the `sidecar` configuration, which was handling the ingress traffic from port `8080`, we can observe that we are no longer able to handle the incoming requests, raising an error message.

```shell
curl 192.168.1.50/helloworld -s
```
```text
upstream connect error or disconnect/reset before headers. reset reason: connection failure, transport failure reason: delayed connect error: 111
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
Error from server (NotFound): error when deleting "Sidecar.yaml": sidecars.networking.istio.io "helloworld-sidecar" not found
```


