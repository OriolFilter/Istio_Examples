---
gitea: none
include_toc: true
---

# Based on

- [07-HTTPS-Gateway-Simple-TLS](../03-HTTPS-Gateway-Simple-TLS)

# Description

This example adds `HTTP` to `HTTPS` redirect at the port `80` of the gateway, which listens for HTTP traffic.

Also contains a listens for `HTTPS` traffic at the port 443, with a self-signed certificate. This will be used to ensure that the redirect is working correctly.


> **NOTE:**\
> This example is kept at minimal, without the need of containing a `Virtual Service`, a `Service` nor a `Deployment/Pod`.

# Configuration applied

## Gateway

The port `80` listens for any host, expecting `HTTP` traffic.\
As `tls.httpsRedirect` is set to `true`, the incoming traffic will be redirected to `HTTPS`, effectively enabling the `HTTP` to `HTTPS` redirect.


The port `443` is expecting traffic that use the `HTTPS` protocol, without being limited to specific hosts.\
The TLS configuration is set to simple, and the credentials (the object that contains the certificates/TLS configuration) is set to `my-tls-cert-secret`.

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
      tls:
        httpsRedirect: true
    - port:
        number: 443
        name: https-web
        protocol: HTTPS
      hosts:
        - "*"
      tls:
        mode: SIMPLE
        credentialName: my-tls-cert-secret
```

> **Note:**\
> The credentials resource is created further bellow through the [Walkthrough](#walkthrough) steps.

> **Note:**\
> For more information regarding the TLS mode configuration, refer to the following [Istio documentation regarding the TLS mode field](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode).

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
gateway.networking.istio.io/helloworld-gateway created
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

### Curl --HEAD

We receive the status message `301 Moved Permanently`.

This points that we are being redirected somewhere. By default, `curl` doesn't follow the redirects. 

To confirm the redirect is being performed towards the `HTTPS` service, we allow `curl` to follow redirects.

```shell
curl  http://192.168.1.50 -I
```
```text
HTTP/1.1 301 Moved Permanently
location: https://192.168.1.50/
date: Tue, 25 Apr 2023 23:59:34 GMT
server: istio-envoy
transfer-encoding: chunked
```

### Curl --HEAD follow redirects

Allowing `curl` to follow redirects, we notice the following output:

- First we receive the same message from before, as we connected to the same service.

- Afterwards we are met with a status code `404`, which is expected as we don't have any service configured behind this gateway.

> **NOTE:**\
> Due using a self-signed certificate, had to allow accessing "insecure" `HTTPS` destinations.  

```shell
curl  http://192.168.1.50 -I -L -k
```
```text
HTTP/1.1 301 Moved Permanently
location: https://192.168.1.50/
date: Wed, 26 Apr 2023 00:05:24 GMT
server: istio-envoy
transfer-encoding: chunked

HTTP/2 404 
date: Wed, 26 Apr 2023 00:05:24 GMT
server: istio-envoy
```

## Cleanup

```shell
kubectl delete -n istio-system secret my-tls-cert-secret
```

```text
secret "my-tls-cert-secret" deleted
```

```shell
kubectl delete -f ./
```
```text
gateway.networking.istio.io "helloworld-gateway" deleted
```

```shell
rm -rv certfolder/
```

```text
removed 'certfolder/istio.cert.key'
removed 'certfolder/istio.cert.crt'
removed directory 'certfolder/'
```

# Links of Interest


- https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings