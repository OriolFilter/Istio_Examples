---
gitea: none
include_toc: true
---

# Description

Based on the previous example where we configured an external service through a `ServiceEntry` object, this example compares the behavior between setting up the MeshConfig `OutboundTrafficPolicy.mode` setting to `REGISTRY_ONLY` and `ALLOW_ANY`.

- ALLOW_ANY: Allows all egress/outbound traffic from the mesh.

- REGISTRY_ONLY: Restricted to services that figure in the service registry a and the ServiceEntry objects.

More info regarding this configuration at the pertinent documentation (https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-OutboundTrafficPolicy-Mode)

> **Note:**\
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-nginx-demo)

# Based on

- [01-Service_Entry](../01-Service_Entry)

# Configuration

## Gateway

Deploys an Istio gateway that's listening to the port `80` for `HTTP` traffic.

It doesn't filter for any specific host.

The `selector` field is used to "choose" which Istio Load Balancers will have this gateway assigned to.

The Istio `default` profile creates a Load Balancer in the namespace `istio-system` that has the label `istio: ingressgateway` set, allowing us to target that specific Load Balancer and assign this gateway resource to it.

```shell
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

> **Note:**\
> The credentials resource is created further bellow through the [Walkthrough](#walkthrough) steps.

> **Note:**\
> For more information regarding the TLS mode configuration, refer to the following [Istio documentation regarding the TLS mode field](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode).

## VirtualService

This configuration hosts 2 backends, 1 being the deployed service `helloworld.default.svc.cluster.local`, which will be accessible through the URL path `/helloworld`.

The second service will be accessible through the URL path `/external`, and will use as a backend the deployed `ServiceEntry` object, as well it has a timeout setting of 3 seconds.

This destination is the service that contains the `HTTPS` deployment, running over the port `8443`

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

    - timeout: 3s
      match:
        - uri:
            exact: "/external"
      route:
        - destination:
            host: help.websiteos.com
            port:
              number: 80
      rewrite:
        uri: "/websiteos/example_of_a_simple_html_page.htm"
      headers:
        request:
          set:
            HOST: "help.websiteos.com"
```

## Service

The service will forward incoming HTTP TCP traffic from the port `80`, towards the deployment port `80`.

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

Nginx deployment listens to port 80.

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

### ServiceEntry

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc
spec:
  hosts:
    - help.websiteos.com
  ports:
    - number: 80
      name: http
      protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
```

## ServiceEntry

This `ServiceEntry` resource, defines as a destination the URL `help.websiteos.com`.

Note that location is set to `MESH_EXTERNAL` and that the resolution is set to `DNS`, this means that the resource is external to ou `Istio Service Mesh`, and the URL will be resolved through `DNS`

Bear in mind that when Istio is communicating with resources externals to the mesh, `mTLS` is disabled.

Also, policy enforcement is performed in the client side instead of the server side.

> **Note:**/
> For more information regarding the `resolution` field or the `location` field, refer to the following official Istio documentations:\
> - [ServiceEntry.Location](https://istio.io/latest/docs/reference/config/networking/service-entry/#ServiceEntry-Location)\
> - [ServiceEntry.Resolution](https://istio.io/latest/docs/reference/config/networking/service-entry/#ServiceEntry-Resolution)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc
spec:
  hosts:
    - help.websiteos.com
  ports:
    - number: 80
      name: http
      protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
```


# Walkthrough

## Set ALLOW_ANY outbound traffic policy

First step will be to have the cluster with the `meshConfig.outboundTrafficPolicy.mode` setting set to `ALLOW_ANY`.

In case you are not using a "free to destroy" sandbox, you should update the setting through the `IstioOperator` object.

```shell
istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout  --set meshConfig.outboundTrafficPolicy.mode=ALLOW_ANY
```

## Deploy resources

```shell
kubectl apply -f ./
```
```text
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
service/helloworld created
serviceentry.networking.istio.io/external-svc created
virtualservice.networking.istio.io/helloworld-vs created
```

## Get LB IP

```shell
kubectl get svc istio-ingressgateway -n istio-system
```

```text
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```

## Test deployments

```shell
curl 192.168.1.50/helloworld -I
```

```text
HTTP/1.1 200 OK
server: istio-envoy
date: Sat, 14 Oct 2023 10:53:45 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 15 Aug 2023 17:03:04 GMT
etag: "64dbafc8-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 53
```

```shell
curl 192.168.1.50/external -I
```

```text
HTTP/1.1 200 OK
date: Sat, 14 Oct 2023 10:54:13 GMT
content-type: text/html
content-length: 5186
last-modified: Mon, 17 Mar 2014 17:25:03 GMT
expires: Thu, 31 Dec 2037 23:55:55 GMT
cache-control: max-age=315360000
x-envoy-upstream-service-time: 306
server: istio-envoy
```


## Test egress the helloworld deployment

It returns a 301 code, meaning that it was able to reach the destination, and it was attempted to redirect the traffic from HTTP to HTTPS.

```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl wikipedia.com -I
```

```text
HTTP/1.1 301 Moved Permanently
server: envoy
date: Sat, 14 Oct 2023 10:54:34 GMT
content-type: text/html
content-length: 169
location: https://wikipedia.com/
x-envoy-upstream-service-time: 61
```

## Set REGISTRY_ONLY outbound traffic policy

```shell
istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout  --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
```

In case you are not using a "free to destroy" sandbox, you should update the setting through the `IstioOperator` object.

## Test (again) egress the helloworld deployment

It returns a 502 code, meaning that it wasn't able to reach the destination.

```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl wikipedia.com -I
```

```text
HTTP/1.1 502 Bad Gateway
date: Thu, 20 Apr 2023 18:08:37 GMT
server: envoy
transfer-encoding: chunked
```

This allowed us to confirm how the setting `outboundTrafficPolicy.mode` influences the reachability of the traffic.

## Cleanup

```shell
kubectl delete -f ./
```
```text
deployment.apps "helloworld-nginx" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
service "helloworld" deleted
serviceentry.networking.istio.io "external-svc" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
```

# Links of Interest

- https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/#controlled-access-to-external-services

- https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/#envoy-passthrough-to-external-services
