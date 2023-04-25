---
gitea: none
include_toc: true
---

# Based on

- [08a-HTTPS-min-TLS-version](../08a-HTTPS-min-TLS-version)

# Description

The previous example was modified to set TCP forwarding towards the backend (HTTP and HTTPS backend).

The backend contains an HTTPS service, which is used to demonstrate how the TCP forwarding is working as intended (aka doesn't disturb HTTP traffic). 

The same backend also contains the same service but running as HTTP, and for such has also been set in the gateway to display both working as intended.

Additionally, the backend used, has HTTP2 enable, which also will be used to confirm that it's working as intended.

> **Note:**\
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-apache-demo)

# Configuration

## Gateway

Gateway been configured to listen both ports `80` and `443` through the TCP protocol, without any host specified. 

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
        name: tcp-1
        protocol: TCP
      hosts:
        - "*"
    - port:
        number: 443
        name: tcp-2
        protocol: TCP
      hosts:
        - "*"
```

## Virtual service

Virtual service have 2 rules that perform the same behavior, on different ports.

The rules will receive the traffic and forward it to the destination service and port. 

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
  tcp:
    - match:
        - port: 80
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8080
    - match:
        - port: 443
      route:
        - destination:
            host:  helloworld.default.svc.cluster.local
            port:
              number: 8443
```

## Service

The service will forward the incoming TCP traffic with port 8080, to the deployment port 80.
The same behavior is applied for the service port 8443, that will be forwarded towards the port 443 from the deployment.

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
    - port: 8080
      name: http-web
      targetPort: 80
      protocol: TCP
    - port: 8443
      name: https-web
      targetPort: 443
      protocol: TCP
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
$ kubectl get svc -l istio=ingressgateway -A
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```

### curl HTTP

```shell
curl http://192.168.1.50  -s -o=/dev/null -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n'
```
```text
http_version: 1.1
status_code: 426
```

#### curl HTTPS

This already confirms that `HTTP2` is working as intended.

```shell
curl https://192.168.1.50  -ks -o=/dev/null -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n' --http1.1
```
```text
http_version: 2
status_code: 200
```

#### Curl HTTP2

The previous example already displayed that `HTTP2` is working as intended.

This example is maintained due being explicitly to confirm the `HTTP2` feature. 

```shell
curl https://192.168.1.50 -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n' --http2 -sk -o=/dev/null
```
```text
http_version: 2
status_code: 200
```

#### Curl HTTP1.1

We can confirm that `HTTP1.1` also works over `TCP forwarding`. 

```shell
curl https://192.168.1.50 -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n' --http1.1 -sk -o=/dev/null
```
```text
http_version: 1.1
status_code: 200
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

- https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSProtocol

- https://stackoverflow.com/a/51279606

- https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-HTTPSettings-H2UpgradePolicy
