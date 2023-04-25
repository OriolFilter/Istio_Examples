---
gitea: none
include_toc: true
---

# Based on

- [08a-HTTPS-min-TLS-version](../08a-HTTPS-min-TLS-version)

# Description

The previous example was modified set the gateway to enable for HTTP2 traffic. 

https://stackoverflow.com/a/59610581


# Changelog

## Gateway

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
        name: secure-http2
        protocol: HTTP2
      hosts:
        - "*"
      tls:
        mode: SIMPLE
        credentialName: my-tls-cert-secret
        minProtocolVersion: TLSV1_2
```

`<text>`

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
### http2
#### Curl HTTP1

```shell
curl 192.168.1.50/helloworld  -s -o=/dev/null -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n' -HHOST:http2.lb --http1.0
```
```text
http_version: 1.1
status_code: 426
```

#### Curl HTTP1.1

```shell
curl 192.168.1.50/helloworld  -s -o=/dev/null -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n' -HHOST:http2.lb --http1.1
```
```text
http_version: 1.1
status_code: 200
```

#### Curl HTTP2

```shell
curl 192.168.1.50/helloworld  -s -o=/dev/null -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n' -HHOST:http2.lb --http2
```
```text
http_version: 1.1
status_code: 200
```

### http1-web

#### Curl HTTP1

```shell
curl 192.168.1.50/helloworld  -s -o=/dev/null -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n' -HHOST:http1.lb --http1.0
```
```text
http_version: 1.1
status_code: 426
```

#### Curl HTTP1.1

```shell
curl 192.168.1.50/helloworld  -s -o=/dev/null -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n' -HHOST:http1.lb --http1.1
```
```text
http_version: 1.1
status_code: 200
```

#### Curl HTTP2

```shell
curl 192.168.1.50/helloworld  -s -o=/dev/null -w 'http_version: %{http_version}\nstatus_code: %{response_code}\n' -HHOST:http1.lb --http2
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
