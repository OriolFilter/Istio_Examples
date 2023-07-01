---
gitea: none
include_toc: true
---

# Based on

- [08a-HTTPS-min-TLS-version](../../03-Gateway_Ingress/04a-HTTPS-min-TLS-version)

# Description

This example contains a backend that serves HTTPS traffic and can be accessed from both `HTTP` and `HTTPS` requests through the gateway resource.  


> **Note:**\
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-nginx-demo)

# Configuration

## Gateway

The gateway is configured to listen to the port `80` for `HTTP` traffic, and to the port `443`  for `HTTPS` traffic.

The TLS configuration is set to `simple`, and the credentials (the object that contains the certificates/TLS configuration) is set to `my-tls-cert-secret`.

Any of the configured ports has limited the hosts. 

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
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "*"
      tls:
        credentialName: my-tls-cert-secret
        mode: SIMPLE
```

> **Note:**\
> The credentials resource is created further bellow through the [Walkthrough](#walkthrough) steps.

> **Note:**\
> For more information regarding the TLS mode configuration, refer to the following [Istio documentation regarding the TLS mode field](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode).

## VirtualService

The rule that contains, will receive traffic from the port `443` and `80`.

This traffic will be directed towards destination of such is the service `helloworld.default.svc.cluster.local`, with port destination 8443.

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
    - name: https-vs
      match:
        - port: 80
        - port: 443
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8443
```

## DestinationRule

This DestinationRule, will interject the traffic destined to the service `helloworld.default.svc.cluster.local` with port `8443`.

As mentioned in the [Virtual Service](#virtualservice) section, the destination is the `HTTPS` service.

By default, the call would be made with `HTTP` protocol, yet, as the destination is an `HTTPS` service, the request would result in the status code `400 Bad Request`, due sending HTTP traffic to an HTTPS service.

To avoid this, we need to specify that the destination handles HTTPS traffic.

By setting the `tls.mode` field with `simple`, it means that there will be an attempt to initialize a TLS handshake. 

> **Note:** 
> For more information about the TLS mode, refer to the [Istio official documentation from the DestinationRule object regarding the TLS mode field](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ClientTLSSettings-TLSmode). 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld
  namespace: default
spec:
    host: helloworld.default.svc.cluster.local
    trafficPolicy:
      portLevelSettings:
        - port:
            number: 8443
          tls:
            mode: SIMPLE
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
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-nginx-demo)

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

## PeerAuthentication

Due to the deployment having an `HTTPS`, and already initializing a TLS termination towards that service, we need to disable the **mTLS** tool for that specific service/deployment.

On the [Destination Rule](#destinationrule) section we set the `tls` to `simple`, meaning that the service is expecting to receive `HTTPS` traffic, if `mTLS` is enabled, it will perform the handshake with the `mTLS` service, instead of with the destination `HTTPS` service.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default-mtls
  namespace: default
spec:
  mtls:
    mode: DISABLE
```

> **Note**:\
> As this configuration is very board, and targets the whole namespace, I would strongly recommend referring to the following example [06-Internal-Authentication/02-target-service-accounts](../../08-AuthorizationPolicy/02-target-service-accounts), which shows how to target service accounts set to resources, limiting the scope of this rule set.

# Walkthrough

## Generate client and server certificate and key files

First step will be to generate the certificate and key files to be able to set them to the Gateway resource.

### Create a folder to store files.

Create the folder to contain the files that will be generated.

```shell
mkdir certfolder
```

### Create a certificate and a private key.

```shell
openssl req -x509 -sha256 -nodes -days 365 -subj '/O=Internet of things/CN=lb.net' -newkey rsa:2048 -keyout certfolder/istio.cert.key -out certfolder/istio.cert.crt
```

The files generated are the following:

```yaml
private-key: certfolder/istio.cert.key
root-certificate: certfolder/istio.cert.crt
```

The information set to the certificate generated is the following:

```yaml
Organization-name: Internet of things
CN: lb.net
```

### Create a TLS secret

At this step we create the tls secret `my-tls-cert-secret` on the namespace `istio-system`.

```shell
kubectl create -n istio-system secret tls my-tls-cert-secret \
  --key=certfolder/istio.cert.key \
  --cert=certfolder/istio.cert.crt
```
```text
secret/my-tls-cert-secret created
```
```text
service/helloworld created
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
```

> **Note:**\
> It's Important that the secret is located in the same namespace as the Load Balancer used. In my case is the `istio-system`, but it will vary based on the environment.

## Deploy resources

```shell
kubectl apply -f ./
```
```text
peerauthentication.security.istio.io/default-mtls created
service/helloworld created
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
destinationrule.networking.istio.io/helloworld created
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
### curl HTTP  gateway

Well, it works as expected.

```shell
curl --insecure 192.168.1.50 -I 
```
```text
HTTP/1.1 200 OK
server: istio-envoy
date: Tue, 25 Apr 2023 04:41:19 GMT
content-type: text/html
content-length: 15
last-modified: Tue, 25 Apr 2023 00:47:17 GMT
etag: "64472315-f"
strict-transport-security: max-age=7200
accept-ranges: bytes
x-envoy-upstream-service-time: 28
```

### curl HTTPS gateway

Well, it works as expected.

```shell
curl --insecure https://192.168.1.50 -I 
```
```text
HTTP/2 200 
server: istio-envoy
date: Tue, 25 Apr 2023 04:42:07 GMT
content-type: text/html
content-length: 15
last-modified: Tue, 25 Apr 2023 00:47:17 GMT
etag: "64472315-f"
strict-transport-security: max-age=7200
accept-ranges: bytes
x-envoy-upstream-service-time: 13
```


## Cleanup

```shell
kubectl delete -f ./
```

```text
peerauthentication.security.istio.io "default-mtls" deleted
service "helloworld" deleted
deployment.apps "helloworld-nginx" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
destinationrule.networking.istio.io "helloworld" deleted
```

# Links of Interest

- https://istio.io/latest/docs/reference/config/networking/gateway/#Gateway

- https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode

- https://istio.io/latest/docs/reference/config/networking/destination-rule/#ClientTLSSettings-TLSmode