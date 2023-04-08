##### https://github.com/istio/istio/tree/master/samples/helloworld

https://istio.io/latest/blog/2017/0.1-canary/


# Simple Hello World

- 1 Service
- 2 Versions

Iterates between the versions without any specific policy. (actually doesn't use the version for anything)

I think that by default uses `RANDOM`.

https://istio.io/latest/docs/reference/config/networking/destination-rule/#TrafficPolicy-PortTrafficPolicy

https://istio.io/latest/docs/reference/config/networking/destination-rule/#LoadBalancerSettings


Relies in automatic sidecar injection.


> Contains service account configurations, yet they are commented as not "necessary".

## Quick note

On this version I have "started" to use the full service name instead of the shorten version, aka:

```yaml
      route:
        - destination:
            host: helloworld
```

Will be:

```yaml
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
```

It's overall a good practice to have, so not much of a reason to not do it.

https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule
 

## Files

- deployment.yaml
- gateway.yaml

## deployment.yaml

### Creates

#### Service

- helloworld

#### Deployments

- helloworld-v1 (Nginx)
- helloworld-v2 (Apache)

## gateway.yaml

### Creates

#### Gateway

##### helloworld-gateway

###### Configuration

```yml
port: 80
istio-ingress: ingressgateway
hosts: "*"
```

#### VirtualService

##### helloworld-vs

###### Configuration

```yaml
hosts: "*"
uri: "/helloworld"
versions:
   v1:
      weight: "25%"
   v2:
     weight: "75%"
```

#### Destination Rule

###### Configuration

```yaml

```


# Run example

## Deploy resources

```shell
$ kubectl apply -f ./ 
service/helloworld created
deployment.apps/helloworld-v1 created
deployment.apps/helloworld-v2 created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
destinationrule.networking.istio.io/helloworld-destinationrule created
```

## Wait for the pods to be ready

(I think it deploys 2 pods as there is the Envoy Proxy pod besides the Nginx deployment)

```shell
$ kubectl get deployment helloworld-v{1..2} -w                                                                                                                                        ✔  kubernetes-admin@kubernetes ⎈ 
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-v1   1/1     1            1           4m1s
helloworld-v2   1/1     1            1           4m1s
```

## Test the service

### Get LB IP

```shell
$ kubectl get svc istio-ingressgateway -n istio-system 
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```

### Curl

Iterates between Nginx and Apache. Somwhat close to the ratio configured.

> Nginx instances (v1): 2 \
> Apache instances (v2): 9

```shell
$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<h1>Welcome to nginx!</h1>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"                                                                                                                                                                         ✔ 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<h1>Welcome to nginx!</h1>
```
