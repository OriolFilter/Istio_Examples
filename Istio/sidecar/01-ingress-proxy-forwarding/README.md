



# Continues from

- 01-hello_world_1_service_1_deployment





---

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
rewrite:
  uri: "/"
```
- Allows the traffic from that have any domain.

- Only allows traffic that has as a destination the directory/path `/helloworld`.

- `rewrite.uri` allows to redirect the traffic towards the root directory of the service, as the service(s) used don't have any directory named `helloworld` but are configured to work at the root base level.

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