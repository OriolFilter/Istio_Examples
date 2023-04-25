---
gitea: none
include_toc: true
---

# Based on

- [08a-HTTPS-min-TLS-version](../08a-HTTPS-min-TLS-version)

# Description

The previous example was modified to limit and specify the maximum TLS version. 

# Changelog

## Gateway

Gateway has been modified to limit the maximum TLS version to v1.2.

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
        number: 443
        name: secure-http
        protocol: HTTPS
      hosts:
        - "*"
      tls:
        mode: SIMPLE
        credentialName: my-tls-cert-secret
        maxProtocolVersion: TLSV1_2
```


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
service/helloworld created
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
```

## Test the service

### Curl TLS 1.2

It fails as intended.

As the TLS v1.2 is smaller than the TLS v1.3 set as a minimal TLS version accepted, it doesn't allow us to proceed with the request.

```shell
curl  --insecure https://192.168.1.50/helloworld -I --tlsv1.2 --tls-max 1.2
```

```text
HTTP/2 200 
server: istio-envoy
date: Sun, 23 Apr 2023 05:48:04 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 7
```

### Curl TLS 1.3

It works as intended due respecting the minimal TLS version set.

```shell
curl  --insecure https://192.168.1.50/helloworld -I --tlsv1.3 --tls-max 1.3
```

```text
curl: (35) OpenSSL/3.0.8: error:0A00042E:SSL routines::tlsv1 alert protocol version
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
service "helloworld" deleted
deployment.apps "helloworld-nginx" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
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

- https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSProtocol

- https://discuss.istio.io/t/minimum-tls-version/5541/3