# Continues from

- 05-hello_world_1_Service_Entry

# Description

On this example compares the behavior between setting up the MeshConfig `OutboundTrafficPolicy.mode` setting to `REGISTRY_ONLY` and `ALLOW_ANY`.

- ALLOW_ANY: Allows all egress/outbound traffic from the mesh.

- REGISTRY_ONLY: Restricted to services that figure in the service registry a and the ServiceEntry objects.

More info regarding this configuration at the pertintent documentation (https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-OutboundTrafficPolicy-Mode)

## Runthrough

### Set ALLOW_ANY outbound traffic policy

```shell
istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout  --set meshConfig.outboundTrafficPolicy.mode=ALLOW_ANY
```

### Deploy resources

```shell
$ kubectl apply -f ./
service/helloworld created
deployment.apps/helloworld-nginx created
serviceentry.networking.istio.io/external-svc created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
```

### Get LB IP

```shell
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```

### Test deployments

```shell
$ curl 192.168.1.50/helloworld -I
HTTP/1.1 200 OK
server: istio-envoy
date: Thu, 20 Apr 2023 18:03:18 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 73
```

```shell
$ curl 192.168.1.50/external -I
HTTP/1.1 200 OK
date: Thu, 20 Apr 2023 18:03:24 GMT
content-type: text/html
content-length: 5186
last-modified: Mon, 17 Mar 2014 17:25:03 GMT
expires: Thu, 31 Dec 2037 23:55:55 GMT
cache-control: max-age=315360000
x-envoy-upstream-service-time: 228
server: istio-envoy
```


### Test egress the helloworld deployment

It returns a 301 code, meaning that it was able to reach the destination and it was attempted to redirect the traffic from HTTP to HTTPS.

```shell
$ kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl wikipedia.com -I
HTTP/1.1 301 Moved Permanently
server: envoy
date: Thu, 20 Apr 2023 18:06:57 GMT
content-type: text/html
content-length: 169
location: https://wikipedia.com/
x-envoy-upstream-service-time: 65
```

### Set REGISTRY_ONLY outbound traffic policy

```shell
istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout  --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
```

### Test (again) egress the helloworld deployment

It returns a 502 code, meaning that it wasn't able to reach the destination.

```shell
$ kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl wikipedia.com -I
HTTP/1.1 502 Bad Gateway
date: Thu, 20 Apr 2023 18:08:37 GMT
server: envoy
transfer-encoding: chunked
```