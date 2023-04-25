---
gitea: none
include_toc: true
---

# Based on

- [10-TCP-FORWARDING](../10-TCP-FORWARDING)

# Description

The previous example was modified set TLS Forwarding for the HTTPS, meaning that the TLS will be terminated by the backend containing a service capable of such.

This requires a deployment with a service HTTPS (as it will need to handle the TLS termination ...). 

> **Note:**\
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-apache-demo)

# Configuration

## Gateway

Gateway configured to listen the port `443` for `HTTPS` traffic protocol.

The tls was configured as `PASSTHROUGH`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: helloworld-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https-web
        protocol: HTTPS
      hosts:
        - "*"
      tls:
        mode: PASSTHROUGH
```

## Virtual service

Virtual service expected to receive traffic with designation, the host `lb.net`.

The rule that contains, will receive traffic from the port `443`, with host destination `lb.net`.

The destination of such is the service `helloworld.default.svc.cluster.local`, with port destination 8443.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-vs
  namespace: default
spec:
  hosts:
    - "lb.net"
  gateways:
    - helloworld-gateway
  tls:
    - match:
        - port: 443
          sniHosts: ["lb.net"]
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8443
```

## Service

The service will forward incoming TCP traffic from the port `8443`, towards the deployment port `443`.

It's been specified the protocol expected to service, it being `HTTPS`.

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
    - name: https
      port: 8443
      targetPort: 443
      protocol: TCP
      appProtocol: HTTPS
  selector:
    app: helloworld
```

## Deployment

Deployment listens to port 80 and 443.

> **Note:**\
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-apache-demo)

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
          image: oriolfilter/https-nginx-demo
          resources:
            requests:
              cpu: "100m"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
            - containerPort: 443
```

# Walkthrough

## Deploy resources

```shell
kubectl apply -f ./
```
```text
service/helloworld created
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
```

## Test the service

### Get LB IP

```shell
kubectl get svc -l istio=ingressgateway -A
```
```text
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```
### curl HTTPS

Well, it just works.

The `--resolve` flag it's used to "fake" the traffic to match the filters we specified in the `Virtual Service`, specifically the `host` and `hostSNI` fields.

```shell
curl --insecure --resolve lb.net:443:192.168.1.50 https://lb.net
```
```text
<h2>Howdy</h2>
```

### curl HTTPS (HEAD)

Here we can spot the following sentence:

- `server: nginx/1.23.4`

This means that the TLS was handled by Nginx (verifying that the `TLS Passthrough` was performed correctly).

If it had been managed by Istio, it would say:

- `server: istio-envoy`

```shell
curl --insecure --resolve lb.net:443:192.168.1.50 https://lb.net --HEAD
```
```text
HTTP/2 200 
server: nginx/1.23.4
date: Tue, 25 Apr 2023 02:49:33 GMT
content-type: text/html
content-length: 15
last-modified: Tue, 25 Apr 2023 00:47:17 GMT
etag: "64472315-f"
strict-transport-security: max-age=7200
accept-ranges: bytes
```

## Cleanup

```shell
kubectl delete -f ./
```

```text
service "helloworld" deleted
deployment.apps "helloworld-nginx" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
```

# Links of Interest

- https://istio.io/latest/docs/reference/config/networking/gateway/#Gateway

- https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode