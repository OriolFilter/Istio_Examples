# Description

This image was intended to be used on configuration tests or troubleshooting.

URL: [`docker.io/oriolfilter/https-nginx-demo:latest`](https://hub.docker.com/r/oriolfilter/https-nginx-demo)

---

## Breakdown

### Capabilities

- Multi arch
- HTTP
- HTTPS (with built-in certificate)
- HTTP2
- Nginx 

### Platforms it was build on:

- linux/amd64
- linux/arm64
- linux/arm/v7

### Dockerfile

The orders given are very simple:

1. Grab the nginx image as a base/template (this allows me to forget about the entrypoint configuration).

2. Take the file `server.conf` and place it in the path `/etc/nginx/conf.d/default.conf` from the container/image.

3. Create the directory `/var/www/html`, and afterwards create a simple index. 

4. Create a certificate and a key that will be used on the Nginx to allow HTTPS traffic requests.

```Dockerfile
FROM nginx

ADD server.conf /etc/nginx/conf.d/default.conf

RUN mkdir -p /var/www/html
RUN echo "<h2>Howdy</h2>" | tee /var/www/html/index.html

RUN openssl req -x509 -sha256 -nodes -days 358000 -subj '/O=SSL EXAMPLE/CN=lb.net' -newkey rsa:2048 -keyout /cert.key -out /cert.crt
```
### server.conf

Read it if you please.

The port listens to both port 80 and port 443, for HTTP and HTTPS traffic.

Port 443 has enabled http2.

Could have configured HTTP to HTTPS forwarding, yet this way I can verify the status of the service or configurations through HTTP requests. (also the HTTP to HTTPS forwarding should be handled by the Load Balancer / Ingress)

It uses the certificates generated previously.

```nginx
server {
  listen 80;

  server_name lb.net;

  access_log /var/log/nginx/access.log;
  error_log  /var/log/nginx/error.log info;

  add_header Strict-Transport-Security "max-age=7200";

  root /var/www/html;
  index index.html;
}

server {
  listen 443 ssl default_server http2;

  ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;

  ssl_ciphers ALL:!aNULL:!ADH:!eNULL:!LOW:!EXP:RC4+RSA:+HIGH:+MEDIUM;

  server_name lb.net;

  access_log /var/log/nginx/access.log;
  error_log  /var/log/nginx/error.log info;

  ssl on;
  ssl_certificate /cert.crt;
  ssl_certificate_key /cert.key;
  ssl_session_timeout  5m;

  add_header Strict-Transport-Security "max-age=7200";

  root /var/www/html;
  index index.html;
}
```

# Build it yourself

[Used this guide through this process](https://docs.docker.com/build/building/multi-platform/)

# Yes

As far I understood, runs this as privileged to install certain packages / architectures / platforms to your device.

```shell
docker run --privileged --rm tonistiigi/binfmt --install all
```
```text
Unable to find image 'tonistiigi/binfmt:latest' locally
latest: Pulling from tonistiigi/binfmt
8d4d64c318a5: Pull complete 
e9c608ddc3cb: Pull complete 
Digest: sha256:66e11bea77a5ea9d6f0fe79b57cd2b189b5d15b93a2bdb925be22949232e4e55
Status: Downloaded newer image for tonistiigi/binfmt:latest
installing: arm OK
installing: mips64le OK
installing: mips64 OK
installing: arm64 OK
installing: riscv64 OK
installing: s390x OK
installing: ppc64le OK
{
  "supported": [
    "linux/amd64",
    "linux/arm64",
    "linux/riscv64",
    "linux/ppc64le",
    "linux/s390x",
    "linux/386",
    "linux/mips64le",
    "linux/mips64",
    "linux/arm/v7",
    "linux/arm/v6"
  ],
  "emulators": [
    "qemu-aarch64",
    "qemu-arm",
    "qemu-mips64",
    "qemu-mips64el",
    "qemu-ppc64le",
    "qemu-riscv64",
    "qemu-s390x"
  ]
}
```

## Create builder profile

```shell
docker buildx create --name mybuilder --driver docker-container --bootstrap
```
```text
[+] Building 2.0s (1/1) FINISHED                                                                                                                                                                                                          
 => [internal] booting buildkit                                                                                                                                                                                                      2.0s
 => => pulling image moby/buildkit:buildx-stable-1                                                                                                                                                                                   1.2s
 => => creating container buildx_buildkit_mybuilder0                                                                                                                                                                                 0.8s
mybuilder
```

## Use created buildx profile

```shell
docker buildx use mybuilder
```

## Inspect selected buildx profile

```shell
docker buildx inspect
```
```text
Name:          mybuilder
Driver:        docker-container
Last Activity: 2023-04-25 00:33:29 +0000 UTC

Nodes:
Name:      mybuilder0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Buildkit:  v0.11.5
Platforms: linux/amd64, linux/amd64/v2, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6
```


## Build, tag and push

I am targeting the repo directly, but any registry can be targeted.

```shell
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 -t oriolfilter/https-nginx-demo:latest . --push
```

```text
[+] Building 11.0s (24/24) FINISHED                                                                                                                                                                                                       
 => [internal] load .dockerignore                                                                                                                                                                                                    0.0s
 => => transferring context: 2B                                                                                                                                                                                                      0.0s
 => [internal] load build definition from Dockerfile                                                                                                                                                                                 0.0s
 => => transferring dockerfile: 383B                                                                                                                                                                                                 0.0s
 => [linux/arm/v7 internal] load metadata for docker.io/library/nginx:latest                                                                                                                                                         0.8s
 => [linux/arm64 internal] load metadata for docker.io/library/nginx:latest                                                                                                                                                          0.8s
 => [linux/amd64 internal] load metadata for docker.io/library/nginx:latest                                                                                                                                                          0.8s
...

<> Building sounds intensifies <>

...
 => [auth] oriolfilter/https-nginx-demo:pull,push token for registry-1.docker.io
```