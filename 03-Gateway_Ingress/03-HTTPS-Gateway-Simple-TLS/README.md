---
gitea: none
include_toc: true
---

# Based on

- [01-hello_world_1_service_1_deployment](../../01-Getting_Started/01-hello_world_1_service_1_deployment)

# Description

On this example, we generate a TLS configuration, and afterwards we attach such to a `Gateway` resource listening to the port `443` for `HTTPS` traffic.

> **Note:** \
> This was based on the information from the following Istio documentation:
> - [Secure Gateways](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/)

# Configuration applied

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
        name: secure-http
        protocol: HTTPS
      hosts:
        - "*"
      tls:
        mode: SIMPLE
        credentialName: my-tls-cert-secret
```

- Gateway is listening to the port `443` and `HTTPS` protocol.

- Allows for all hosts.

- The TLS configuration is set to simple, and the credentials (the object that contains the certificates/TLS configuration) is set to `my-tls-cert-secret`.

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
service/helloworld created
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
```

## Test the service

[//]: # (```shell)
[//]: # (curl --insecure --resolve lb.net:443:192.168.1.50 https://lb.net/helloworld)
[//]: # (```)

```shell
curl  --insecure https://192.168.1.50/helloworld -I
```

```text
HTTP/2 200 
server: istio-envoy
date: Sun, 23 Apr 2023 05:06:47 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 96
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

# Troubleshooting.

## curl: (7) Failed to connect to 192.168.1.51 port 443 after 2 ms: Couldn't connect to server

- Ensure that the gateway is listening to the right port, in this case, the port 443.

- Refer to the troubleshooting documentation, specifically the `Logs>Ingress`. \
Check if it displays any log activity that could facilitate the troubleshooting / investigation.

## curl: (35) Recv failure: Connection reset by peer

- Refer to the troubleshooting documentation, specifically the `Logs>Ingress`. \
  Check if it displays any log activity that could facilitate the troubleshooting / investigation.

## 404

Ensure the URL used to thest the connectivity, matches the host and path rules applied, both in the `Gateway` and `VirtualService` resources.

# Links of Interest

- https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress

- https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode