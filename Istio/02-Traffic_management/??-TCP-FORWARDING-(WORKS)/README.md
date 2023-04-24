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

- https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSProtocol

- https://stackoverflow.com/a/51279606

- https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-HTTPSettings-H2UpgradePolicy



docker buildx build --push --platform linux/arm/v7,linux/arm64/v8,linux/amd64 --tag registery.filter.home:5000/https-demo:latest -f Dockerfile


docker buildx build --push --platform linux/arm/v7,linux/arm64/v8,linux/amd64 --tag registery.filter.home:5000/https-demo:latest .            
[+] Building 0.0s (0/0)                                                                                                                                                                                                                   
ERROR: multiple platforms feature is currently not supported for docker driver. Please switch to a different driver (eg. "docker buildx create --use")

---
## Create the Dockerfile

```bash
FROM ubuntu/apache2

RUN apt-get update && \
apt-get install apache2 openssl -y && \
a2ensite default-ssl && \
a2enmod ssl && \
echo "<h2>Howdy</h2>" | tee /var/www/html/index.html

RUN /usr/bin/printf "<VirtualHost *:80>\n\
	ServerAdmin webmaster@localhost\n\
	DocumentRoot /var/www/html\n\
	ErrorLog \${APACHE_LOG_DIR}/error.log\n\
	CustomLog \${APACHE_LOG_DIR}/access.log combined\n\
</VirtualHost>\n\
<VirtualHost *:443>\n\
    ServerAdmin webmaster@localhost\n\
    DocumentRoot /var/www/html\n\
    ErrorLog \${APACHE_LOG_DIR}/error.log\n\
    CustomLog \${APACHE_LOG_DIR}/access.log combined\n\
    SSLEngine on\n\
    SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem\n\
    SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key\n\
</VirtualHost>" > /etc/apache2/sites-available/000-default.conf

RUN openssl req -x509 -sha256 -nodes -days 358000 -subj '/O=SSL EXAMPLE' -newkey rsa:2048 -keyout /etc/ssl/private/ssl-cert-snakeoil.key -out /etc/ssl/certs/ssl-cert-snakeoil.pem
```

## Build the image

Due to my Kubernetes cluster environment, where I am using Orange 5, their architecture is arm7, and for such, I require to compile such images.

For my own commodity, I have used a raspberry pi 4 to build this images.

The images where pushed to a local registry server, and afterwards the Kubernetes cluster will pull such image.

```shell
 docker build --tag https-demo:armv7  .
```
```text
docker build --tag https-demo:armv7 . --no-cache
[+] Building 16.5s (8/8) FINISHED                                                                                  
 => [internal] load .dockerignore                                                                             0.0s
 => => transferring context: 2B                                                                               0.0s
 => [internal] load build definition from Dockerfile                                                          0.0s
 => => transferring dockerfile: 1.09kB                                                                        0.0s
 => [internal] load metadata for docker.io/ubuntu/apache2:latest                                              0.4s
 => CACHED [1/4] FROM docker.io/ubuntu/apache2@sha256:0a5e7179fa8fccf17843a8862e58ac783628b7d448cd68fda8fb1e  0.0s
 => [2/4] RUN apt-get update && apt-get install apache2 openssl -y && a2ensite default-ssl && a2enmod ssl &  12.0s
 => [3/4] RUN /usr/bin/printf "<VirtualHost *:80>\n ServerAdmin webmaster@localhost\n DocumentRoot /var/www/  0.7s 
 => [4/4] RUN openssl req -x509 -sha256 -nodes -days 358000 -subj '/O=SSL EXAMPLE' -newkey rsa:2048 -keyout   2.4s 
 => exporting to image                                                                                        1.0s 
 => => exporting layers                                                                                       1.0s 
 => => writing image sha256:591c6d233100a48bf132eef7a792942cfd0b7057817c4ac5e156c1d33e24cd89                  0.0s 
 => => naming to docker.io/library/https-demo:armv7                                                           0.0s                                             
```

## Tag the image

```shell
docker image tag https-demo:armv7 registery.filter.home/https-demo:armv7
```

## Upload to the registery server

```text
docker image push registery.filter.home:5000/https-demo:armv7
The push refers to repository [registery.filter.home:5000/https-demo]
c6d858706b08: Pushed 
9e077e0202f0: Pushed 
6ffc708d0cf3: Pushed 
69e01b4bf4d7: Pushed 
17c5b30f3843: Pushed 
0b9f60fbcaf1: Pushed 
armv7: digest: sha256:d8c81c27f23bf3945ae8a794c82182f9e6c48ec927f388fdf4a88caa0e284bd1 size: 1578
```



## ?
curl: (35) OpenSSL/3.0.8: error:0A00010B:SSL routines::wrong version numbe





---


Has apache2 installed with a default certificate.

Port 80 visible for HTTP

Port 443 visible for HTTPS.