# Continues from

- 01-hello_world_1_service_1_deployment

# TO TRAFFIC PATH DIAGRAM    etc -> "POD" -> sidecar -> service container

# Description

This example configures the sidecar proxy on the pods created, to forward the traffic incoming from the port `8080` to the port `80`

## Files

- deployment.yaml
- gateway.yaml
- sidecar.yaml

> Added the `sidecar.yaml` file.

## deployment.yaml

### Creates

#### Service

- helloworld

#### Deployments

- helloworld-nginx  (Nginx container)

## gateway.yaml

### Creates

#### Gateway

##### helloworld-gateway

###### Configuration

```yml
...
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

#### VirtualService

##### helloworld-vs

###### Configuration

```yaml
...
spec:
  hosts:
    - "*"
  gateways:
    - helloworld-gateway
  http:
    - match:
        - uri:
            exact: /helloworld
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8080
      rewrite:
        uri: "/"
```

- On this example, we are using the port `8080` as a destination.

## sidecar.yaml

### creates

#### sidecar

##### helloworld-sidecar

###### Configuration

```yaml
...
spec:
  workloadSelector:
    labels:
      app: helloworld
  ingress:
    - port:
        number: 8080
        protocol: HTTP
        name: ingressport
      defaultEndpoint: 127.0.0.1:80
````

workloadSelector:

> `workloadSelector` is used to target the `PODS`, on which apply this sidecar configuration. \
> Bear in mind that this configuration doesn't target kinds `Service`, nor `Deployment`, it's applied to a kind `Pod` or `ServiceEntry` \
> If there is no `workloadSelector` specified, it will be used as default configuration for the namespace on which was created. \
> More info in the [Istio documentation for workloadSelector](https://istio.io/latest/docs/reference/config/networking/sidecar/#WorkloadSelector)

ingress:

> Configure the behavior of the ingress traffic.\
> On this "grabs"/targets the ingress traffic with port 8080, and forwards it to the port IP `127.0.0.1` (loopback) respective to the destination pod, with the destination port set to 80, which is the port that the service is currently listening to.

# Run example

## Deploy resources

```shell
$ kubectl apply -f ./
service/helloworld created
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
sidecar.networking.istio.io/helloworld-sidecar created
```

## Wait for the pods to be ready

```shell
$ kubectl get deployment helloworld-nginx -w
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-nginx   1/1     1            1           39s
```

## Test the service

### Get LB IP

```shell
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```

### Curl

```shell
$ curl 192.168.1.50/helloworld -s | grep "<title>.*</title>"
<title>Welcome to nginx!</title>
```

### Delete the sidecar configuration to force failure.


```shell
$ kubectl delete sidecars.networking.istio.io helloworld-sidecar
sidecar.networking.istio.io "helloworld-sidecar" deleted
```
### Curl again

```shell
$ curl 192.168.1.50/helloworld -s
upstream connect error or disconnect/reset before headers. reset reason: connection failure, transport failure reason: delayed connect error: 111
```

