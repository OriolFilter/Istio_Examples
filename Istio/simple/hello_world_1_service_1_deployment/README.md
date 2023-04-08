##### https://github.com/istio/istio/tree/master/samples/helloworld

# Simple Hello World

- 1 Service
- 1 Deployment

I think that by default uses `RANDOM`.

https://istio.io/latest/docs/reference/config/networking/destination-rule/#TrafficPolicy-PortTrafficPolicy

https://istio.io/latest/docs/reference/config/networking/destination-rule/#LoadBalancerSettings


Relies in automatic sidecar injection.


> Contains service account configurations, yet they are commented as not "necessary".
 

## Files

- deployment.yaml
- gateway.yaml

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
```

# Run example

## Deploy resources

```shell
$ kubectl apply -f ./ 
service/helloworld created
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
```

## Wait for the pods to be ready

(I think it deploys 2 pods as there is the Envoy Proxy pod besides the Nginx deployment)

```shell
$ kubectl get deployment helloworld-nginx -w 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-nginx   1/1     1            1           44s
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
$ curl 192.168.1.50/helloworld -s | grep "<title>.*</title>"                                                                                                                                                                   ✔ 
<title>Welcome to nginx!</title>
```