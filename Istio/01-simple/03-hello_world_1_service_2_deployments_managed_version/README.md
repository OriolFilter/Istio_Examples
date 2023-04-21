##### https://github.com/istio/istio/tree/master/samples/helloworld

https://istio.io/latest/blog/2017/0.1-canary/


# Continues from

- 01-hello_world_1_service_1_deployment

# Simple Hello World

- 1 Service
- 2 Versions

Iterates between the versions without any specific policy. (actually doesn't use the version for anything)


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
 

# Changes

## File

- deployment.yaml
- gateway.yaml

> Files used maintains from the last version

## deployment.yaml

### Creates

#### Service

- helloworld

> Service used maintains from the last version

#### Deployments

- helloworld-v1 (Nginx)
- helloworld-v2 (Apache)

> Renamed the old deployment from `helloworld-nginx` to `helloworld-v1`.\
> Created a secondary deployment using apache named `helloworld-v2`.

## gateway.yaml

#### VirtualService

##### helloworld-vs

###### Configuration



```yaml
...
  http:
    - match:
        - uri:
            exact: /helloworld
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
            subset: v1
          weight: 20
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 80
            subset: v2
          weight: 80
...
```

> Distributed the traffic between 2 versions (`subsets`), setting a `25%` to the subset `v1` and a `75%` to the subset `v2`.

> As previously mentioned, the section `http.route.host` points to `helloworld.default.svc.cluster.local`, which is the service we created, on the `default` namespace.




#### Destination Rule

###### Declaration configuration

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld.default.svc.cluster.local # Destination that will "interject"
```

> Here we need to put the `path/destination/service` that we want this rule to interject and manage.

###### Traffic Configuration

```yaml
  host: helloworld.default.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

> On the `Destination Rule` declared the subsets. Each subset has different labels. This will be used to select the deployments within the destination service.

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
$ kubectl get deployment helloworld-v{1..2} -w 
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
$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<h1>Welcome to nginx!</h1>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>" 
<html><body><h1>It works!</h1></body></html>

$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<h1>Welcome to nginx!</h1>
```

## Check istio configs

```sh
$ istioctl x describe pod `kubectl get pod -l app=helloworld,version=v1 -o jsonpath='{.items[0].metadata.name}'`
Pod: helloworld-v1-7454b56b86-4cksf
   Pod Revision: default
   Pod Ports: 80 (helloworld), 15090 (istio-proxy)
--------------------
Service: helloworld
   Port: http 80/HTTP targets pod port 80
DestinationRule: helloworld for "helloworld.default.svc.cluster.local"
   Matching subsets: v1
      (Non-matching subsets v2)
   No Traffic Policy
--------------------
Effective PeerAuthentication:
   Workload mTLS mode: PERMISSIVE


Exposed on Ingress Gateway http://192.168.1.50
VirtualService: helloworld-vs
   Weight 20%
   /helloworld
```


```shell
$ istioctl x describe pod `kubectl get pod -l app=helloworld,version=v2 -o jsonpath='{.items[0].metadata.name 
Pod: helloworld-v2-64b5656d99-5bwgr
   Pod Revision: default
   Pod Ports: 80 (helloworld), 15090 (istio-proxy)
--------------------
Service: helloworld
   Port: http 80/HTTP targets pod port 80
DestinationRule: helloworld for "helloworld.default.svc.cluster.local"
   Matching subsets: v2
      (Non-matching subsets v1)
   No Traffic Policy
--------------------
Effective PeerAuthentication:
   Workload mTLS mode: PERMISSIVE


Exposed on Ingress Gateway http://192.168.1.50
VirtualService: helloworld-vs
   Weight 80%
   /helloworld
```