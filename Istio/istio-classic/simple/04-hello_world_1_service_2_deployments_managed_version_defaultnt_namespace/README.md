##### https://github.com/istio/istio/tree/master/samples/helloworld

https://istio.io/latest/blog/2017/0.1-canary/


# Simple Hello World

- 1 Service
- 2 Versions

Iterates between the versions without any specific policy. (actually doesn't use the version for anything)

I think that by default uses `RANDOM`.

https://istio.io/latest/docs/reference/config/networking/destination-rule/#TrafficPolicy-PortTrafficPolicy

https://istio.io/latest/docs/reference/config/networking/destination-rule/#LoadBalancerSettings


Manually allows the sidecar injection through the label in the pod


https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#controlling-the-injection-policy

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
      weight: "50%"
   v2:
     weight: "50%"
```

#### Destination Rule

###### Configuration

```yaml
host: helloworld.defaultnt.svc.cluster.local # Full destination service, lil better for consistency
subsets:
- name: v1
  labels:
    version: v1
- name: v2
  labels:
    version: v2
```


# Run example

## Deploy resources

```shell
$ 
```

## Wait for the pods to be ready

(I think it deploys 2 pods as there is the Envoy Proxy pod besides the Nginx deployment)

```shell

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
$ curl 192.168.1.50/helloworld -s | grep "<h1>.*</h1>"
<html><body><h1>It works!</h1></body></html>
```