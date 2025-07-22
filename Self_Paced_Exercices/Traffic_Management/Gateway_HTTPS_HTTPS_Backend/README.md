
# DOESN'T WORK

I'm no longer able to set it up.

# Description

You have a backend that uses HTTPS.

Add a Gateway that uses HTTPS and has its own certificate.

The cert/secret to add/create should be the following:

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

```shell
curl 
```