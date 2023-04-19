##### https://github.com/istio/istio/tree/master/samples/helloworld

### Base simple template

# Simple Hello World

- 1 Service
- 1 Deployment

I think that by default uses `RANDOM`.

https://istio.io/latest/docs/reference/config/networking/destination-rule/#TrafficPolicy-PortTrafficPolicy

https://istio.io/latest/docs/reference/config/networking/destination-rule/#LoadBalancerSettings

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
            host: helloworld
            port:
              number: 80
      rewrite:
        uri: "/"
```
- Allows the traffic that have as a destination any domain.

- Only allows traffic that has as a destination the directory/path `/helloworld`.

- `rewrite.uri` allows to redirect the traffic towards the root directory of the service, as the service(s) used don't have any directory named `helloworld` but are configured to work at the root base level.

- Traffic request is sent to the service named `helloworld`, to the service port 80.

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