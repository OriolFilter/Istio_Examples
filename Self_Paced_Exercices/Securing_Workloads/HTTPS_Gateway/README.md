# Description

Create a Gateway that uses HTTP and has a certificate attached to it.

The port 80 should be redirected to the port 443 from the Gateway.

Set up the minimum TLS to be TLS version 1.1.

> **Warning**:\
> TLS version below TLSV1_2 require setting compatible ciphers as by default they no longer include compatible ciphers.

The Gateway object has to be located in the namespace `default`, and be named `generic-https`.

To test follow [the testing section](#test).

## Create the certificate as a secret for the domain `lb.net`:

```shell
mkdir certfolder
```

```shell
openssl req -x509 -sha256 -nodes -days 365 -subj '/O=Internet of things/CN=lb.net' -newkey rsa:2048 -keyout certfolder/istio.cert.key -out certfolder/istio.cert.crt
```

```shell
kubectl create -n istio-ingress secret tls my-tls-cert-secret \
  --key=certfolder/istio.cert.key \
  --cert=certfolder/istio.cert.crt
```

## Test

### Backend/Certificate

With `-I`/ sending a `HEAD` request we can see that we initially received a 301/redirect response, and after following the redirect we received the backend response.

```shell
curl --insecure --resolve lb.net:80:172.18.10.10 --resolve lb.net:443:172.18.10.10 http://lb.net -LI 
```

```text
HTTP/1.1 301 Moved Permanently
location: https://lb.net/
date: Tue, 22 Jul 2025 19:13:03 GMT
server: istio-envoy
transfer-encoding: chunked

HTTP/2 200 
server: istio-envoy
date: Tue, 22 Jul 2025 19:13:03 GMT
content-type: text/html
content-length: 15
last-modified: Tue, 25 Apr 2023 00:47:17 GMT
etag: "64472315-f"
strict-transport-security: max-age=7200
accept-ranges: bytes
x-envoy-upstream-service-time: 0
```

```shell
curl --insecure --resolve lb.net:80:172.18.10.10 --resolve lb.net:443:172.18.10.10 http://lb.net -L
```

```text
<h2>Howdy</h2>
```

### Test the minimum TLS version specified

When setting the max tls at 1.1 doesn't work. When setting it to 1.2 or 1.3 it does work.

```shell
curl --insecure --resolve lb.net:443:172.18.10.10 https://lb.net -L --tls-max 1.1
```

```text
curl: (35) TLS connect error: error:0A0000BF:SSL routines::no protocols available
```

```shell
curl --insecure --resolve lb.net:443:172.18.10.10 https://lb.net -L --tls-max 1.2
```

```text
<h2>Howdy</h2>
```