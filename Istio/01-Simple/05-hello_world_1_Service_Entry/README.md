# Description

This example uses a resource `ServiceEntry` to "integrate" external resources into our `Istio Service Mesh`.

It also explores the different behaviors between specifying the destination URL on the headers or not.

The following page has been used for testing purposes:

- info.cern.ch

> **Quick disclaimer**:\
> I have no relation with that page.

# Configuration

## ServiceEntry

This `ServiceEntry` resource, defines as a destination the URL `info.cern.ch`.

Note that location is set to `MESH_EXTERNAL` and that the resolution is set to `DNS`, this means that the resource is external to ou `Istio Service Mesh`, and the URL will be resolved through `DNS`

Bear in mind that when Istio is communicating with resources externals to the mesh, `mTLS` is disabled.

Also, policy enforcement is performed in the client side instead of the server side.

> **Note:**/
> For more information regarding the `resolution` field or the `location` field, refer to the following official Istio documentations:
> [ServiceEntry.Location](https://istio.io/latest/docs/reference/config/networking/service-entry/#ServiceEntry-Location)
> [ServiceEntry.Resolution](https://istio.io/latest/docs/reference/config/networking/service-entry/#ServiceEntry-Resolution)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-cern-service
spec:
  hosts:
    - info.cern.ch
  ports:
    - number: 80
      name: http
      protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
```

## Gateway

Listens for `HTTP` traffic at the port `80` without limiting to any host.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: helloworld-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```


## VirtualService

There has been configured 2 paths:

- "/external"
- "/external-noh"

Both routes will forward the request towards the destination URL `info.cern.ch`.

Highlight that the destination is `info.cern.ch`, which is the same as the contents set on the field `host` from the [ServiceEntry resource configured above](#serviceentry).

The difference between `/external` and `/external-noh` is that the first path will contain a header named `HOST`, with the contents set to `info.cern.ch`, it being the URL from the external service.

On the [Walkthrough](#walkthrough) section we will observe the different behaviors of these paths, being the only difference the header attributed.

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
    - name: http-external-service
      timeout: 3s
      match:
        - uri:
            exact: "/external"
      route:
        - destination:
            host: info.cern.ch
            port:
              number: 80
      rewrite:
        uri: "/"
      headers:
        request:
          set:
            HOST: "info.cern.ch"

    - name: https-external-service-without-headers
      timeout: 3s
      match:
        - uri:
            exact: "/external-noh"
      route:
        - destination:
            host: info.cern.ch
            port:
              number: 80
      rewrite:
        uri: "/"
```

# Walkthrough

## Deploy the resources

```shell
kubectl apply -f ./
```
```text
serviceentry.networking.istio.io/external-cern-service created
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

### /external

We can visualize the page contents without issues, nothing to highlight.

```shell
curl 192.168.1.50/external
```
```text
<html><head></head><body><header>
<title>http://info.cern.ch</title>
</header>

<h1>http://info.cern.ch - home of the first website</h1>
<p>From here you can:</p>
<ul>
<li><a href="http://info.cern.ch/hypertext/WWW/TheProject.html">Browse the first website</a></li>
<li><a href="http://line-mode.cern.ch/www/hypertext/WWW/TheProject.html">Browse the first website using the line-mode browser simulator</a></li>
<li><a href="http://home.web.cern.ch/topics/birth-web">Learn about the birth of the web</a></li>
<li><a href="http://home.web.cern.ch/about">Learn about CERN, the physics laboratory where the web was born</a></li>
</ul>
</body></html>
```

### /external-noh

We don't receive any output.

This could be due, even if we resolve the destination IP for the URL `info.cern.ch`, the destination might have a Reverse Proxy or any other ingress resource that could condition handling this request.

Due to the `HOST` field not being modified after we set the request, it might not be able to pass the filtering set, weather it is security wise, for example, requiring such field to allow the request; or it being a routing condition, which due not having this field specified, it's not able to route the request towards the destination desired.

```shell
curl 192.168.1.50/external
```
```text
```

## Cleanup

```shell
kubectl delete -f ./
```
```text
serviceentry.networking.istio.io "external-cern-service" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
```

# Links of interest:

- https://istio.io/latest/docs/reference/config/networking/service-entry/#ServiceEntry-Location

