---
gitea: none
include_toc: true
---

# Description

This example deploys the same infrastructure as the [previous example](../../01-Getting_Started/01-hello_world_1_service_1_deployment), configures the **sidecar** `envoy-proxy`/`istio-proxy`/`sidecar-proxy` on the pods created, to limit the egress resources to which the `istio-proxy`, who proxies the traffic from the pod (both ingress and egress), can send request to.

This will be done through 2 principles: <FILL>

This example configures:

    Generic Kubernetes resources:
    - 2 Services
    - 2 Deployments
    - 1 Namespace

    Istio resources:
    - 2 Sidecar configrations

# Based on

- [01-hello_world_1_service_1_deployment](../../01-Getting_Started/01-hello_world_1_service_1_deployment)

# Configuration

## Namespace

Creates a namespace named `foo` with the `istio-proxy` injection enabled.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: foo
  labels:
    istio-injection: "enabled"
```

## Service

### hellowolrd (default/foo namespace)

Creates two services named `helloworld`, one in the namespace `default`, and another in the namespace `foo`.

This service listens for the port `8080` expecting `HTTP` traffic and will forward the incoming traffic towards the port `80` from the destination pod.
Also listens for the port `80` expecting `HTTP` traffic and will forward the incoming traffic towards the port `80` from the destination pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
  namespace: foo
spec:
  ports:
    - port: 8080
      name: http-a
      targetPort: 80

    - port: 80
      name: http-b
      targetPort: 80

  selector:
    app: helloworld
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
  namespace: default
spec:
  ports:
    - port: 8080
      name: http-a
      targetPort: 80

    - port: 80
      name: http-b
      targetPort: 80

  selector:
    app: helloworld
```

## Deployment 

Creates two deployments named `helloworld`, one in the namespace `default`, and another in the namespace `foo`

### helloworld-default

Contains a Nginx server that listens for the port `80`.

It's created in the namespace `default`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-default
  labels:
    app: helloworld
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          image: nginx
          resources:
            requests:
              cpu: "100m"
          imagePullPolicy: IfNotPresent #Always
          ports:
            - containerPort: 80
```


### helloworld-foo

Contains a Nginx server that listens for the port `80`.

It's created in the namespace `foo`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-foo
  labels:
    app: helloworld
  namespace: foo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          image: nginx
          resources:
            requests:
              cpu: "100m"
          imagePullPolicy: IfNotPresent #Always
          ports:
            - containerPort: 80
```


## Sidecar

This will configure the sidecar configuration from the `envoy-proxy` in each pod.

`workloadSelector` will be used to select the target pods, where, on this scenario, it will target the pods that have the label set `app: helloworld`.

> **Note:**\
> A reminder that a `POD` is an object that groups container(s).

+ more notes:

- workloadSelector:

> `workloadSelector` is used to target the `PODS`, on which apply this sidecar configuration. \
> Bear in mind that this configuration doesn't target kinds `Service`, nor `Deployment`, it's applied to a kind `Pod` or `ServiceEntry` \
> If there is no `workloadSelector` specified, it will be used as default configuration for the namespace on which was created. \
> More info in the [Istio documentation for workloadSelector](https://istio.io/latest/docs/reference/config/networking/sidecar/#WorkloadSelector)

- egress:

> Configure the behavior of the proxied egress traffic.\
> On this example, we limit port that the `sidecar-proxy` will be allowed to send traffic to, as well limiting the routes that can the `sidecar-proxy` container will be able to learn the routes from.\
> A reminder that Istio automatically creates routes for each one of the services and each one of the ports configured to be exposed.\
> More info in the [Istio documentation for IstioEgressListener](https://istio.io/latest/docs/reference/config/networking/sidecar/#IstioEgressListener)

- outboundTrafficPolicy.mode:

> The most important step from this configuration.\
> By setting the value to `REGISTRY_ONLY`, it will restrict the egress connectivity towards the destinations defined in the registry as well of the defined `ServiceEntry` configurations.
> Taking into account that the field `egress`, where we limited the routes that the `sidecar-proxy` would be allowed to learn routes from, combined with this setting set to `REGISTRY_ONLY`, we limit the egress reachability from the PODS.\
> If the setting is set to `ALLOW_ANY`, the egress limitation will be ignored.
> More info in the [Istio documentation for OutboundTrafficPolicy.Mode](https://istio.io/latest/docs/reference/config/networking/sidecar/#OutboundTrafficPolicy-Mode)

### helloworld-sidecar-default

On this example we target the Deployments from the namespace `default` that contain a label named `app` with the contents set to `helloworld`.

We limit the egress to the port `80`, and will only be able to reach out to the learned destinations from the namespaces `foo`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: helloworld-sidecar-default
  namespace: default
spec:
  workloadSelector:
    labels:
      app: helloworld
  egress:
    - port:
        number: 80
        protocol: HTTP
        name: egress-http
      hosts:
        - "foo/*"
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
```

### helloworld-sidecar-foo

On this example we target the Deployments from the namespace `foo` that contain a label named `app` with the contents set to `helloworld`.

We limit the egress to the port `8080`, and will only be able to reach out to the learned destinations from the namespaces `default`, and it's own (`./*`) aka. `foo`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: helloworld-sidecar-foo
  namespace: foo
spec:
  workloadSelector:
    labels:
      app: helloworld
  egress:
    - port:
        number: 8080
        protocol: HTTP
        name: egress-default
      hosts:
        - "./*"
        - "default/*"
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
```

# Run example

## Deploy resources

```shell
kubectl apply -f ./
```

```text
namespace/foo created
deployment.apps/helloworld-default created
deployment.apps/helloworld-foo created
service/helloworld created
service/helloworld created
sidecar.networking.istio.io/helloworld-sidecar-default created
sidecar.networking.istio.io/helloworld-sidecar-foo created
```

## Wait for the pods to be ready

```shell
watch -n 5 "kubectl get deployment -A | grep helloworld"
```

```text
default          helloworld-default        1/1     1            1           10s
foo              helloworld-foo            1/1     1            1           10s
```

## Test the service

### from `helloworld-default`

Reminder of the **egress** criteria that has been configured to be met:

[ ] Port `80`.

[ ] `HTTP` protocol.

[ ] Namespace destination `foo`.

#### Curl helloworld.foo.svc.cluster.local:80

On this scenario we meet the following criteria:

[x] Port `80`.

[x] `HTTP` protocol.

[x] Namespace destination `foo`.

```shell
NAMESPACE="default" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.foo.svc.cluster.local:80 -sI
```

```text
HTTP/1.1 200 OK
server: envoy
date: Mon, 15 May 2023 11:49:34 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 10
```





#### Curl helloworld.foo.svc.cluster.local:8080

[ ] Port `80`.

[x] `HTTP` protocol.

[x] Namespace destination `foo`.


```shell
NAMESPACE="default" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.foo.svc.cluster.local:8080 -sI
```

```text
command terminated with exit code 56
```

##### What's happening?

Let's observe the logs activity from the `istio-proxy` container, of the deployment `helloworld` in the namespace `default` when we send request towards the service `helloworld` in the namespace `foo` through the port `8080`.

```shell
NAMESPACE="default" && kubectl logs -l app=helloworld --follow -c istio-proxy -n $NAMESPACE --tail 0
```

From another `shell` send a request towards the destination.

```shell
NAMESPACE="default" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.foo.svc.cluster.local:8080 -sI
```

We can see, how the `istio-proxy` container, from the `helloworld` POD, in the namespace `default`, generates the following log entry:

```text
[2023-05-15T12:19:03.577Z] "- - -" 0 UH - - "-" 0 0 0 - "-" "-" "-" "-" "-" BlackHoleCluster - 10.107.249.242:8080 172.17.247.52:58820 - -
```

On the log generated, it specifies the word `BlackHoleCluster`.

`BlackHoleCluster` is an Istio resource/destination used to block requests, meaning that our request was forwarded to it, preventing us to reach to the desired destination, as per configured in the [sidecar configuration](#sidecar).

I understand that this behavior is caused due that the namespace `foo` is an external location respective to the deployment, and for such it requires `istio-proxy` to learn its destination, whereas in this scenario, due [sidecar configuration](#sidecar), doesn't figure either in the list of accepted routes.

For such, instead the is sent towards `BlackHoleCluster`.





#### Curl helloworld.default.svc.cluster.local:80

[x] Port `80`.

[x] `HTTP` protocol.

[ ] Namespace destination `foo`.


```shell
NAMESPACE="default" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.default.svc.cluster.local:80 -sI
```

```text
HTTP/1.1 502 Bad Gateway
date: Mon, 15 May 2023 12:23:12 GMT
server: envoy
transfer-encoding: chunked
```

##### What's happening?

Let's observe the logs activity from the `istio-proxy` container, of the deployment `helloworld` in the namespace `default` when we send request towards the service `helloworld` in the namespace `default` through the port `80`.

```shell
NAMESPACE="default" && kubectl logs -l app=helloworld --follow -c istio-proxy -n $NAMESPACE --tail 0
```

From another `shell` send a request towards the destination.

```shell
NAMESPACE="default" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.default.svc.cluster.local:80 -sI
```

We can see, how the `istio-proxy` container, from the `helloworld` POD, in the namespace `default`, generates the following log entry:

```text
[2023-05-15T12:24:40.757Z] "HEAD / HTTP/1.1" 502 - direct_response - "-" 0 0 0 - "-" "curl/7.74.0" "952652df-7761-4f15-be58-776eeedfb6cf" "helloworld.default.svc.cluster.local" "-" - - 10.108.186.1:80 172.17.247.52:57516 - block_all
```

On the log generated, we can observe further information than the previous one, nevertheless I want to put emphasis on the following sections:

- `502 - direct_response`

This means that the status code `502` was a `direct response`, coming from istio itself, directly targeting this request.

- `block_all`

Istio already acknowledges this request and flags is as doesn't meet the requirements configured in the [sidecar configuration](#sidecar).

I understand that this behavior is different from when sending a request to `foo` on the port `8080`, in the current configuration set, we didn't specify any egress setting that allow any kind of egress towards the port `80`.

For such it raises a `direct response` with status code `502`, as the `istio-proxy` strictly won't accept any egress request with that port.




#### Curl helloworld.default.svc.cluster.local:8080

[x] Port `8080`.

[x] `HTTP` protocol.

[ ] Namespace destination `foo`.


```shell
NAMESPACE="default" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.default.svc.cluster.local:8080 -sI
```

```text
command terminated with exit code 56
```

##### What's happening?

Let's observe the logs activity from the `istio-proxy` container, of the deployment `helloworld` in the namespace `default` when we send request towards the service `helloworld` in the namespace `default` through the port `8080`.

```shell
NAMESPACE="default" && kubectl logs -l app=helloworld --follow -c istio-proxy -n $NAMESPACE --tail 0
```

From another `shell` send a request towards the destination.

```shell
NAMESPACE="default" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.default.svc.cluster.local:8080 -sI
```

We can see, how the `istio-proxy` container, from the `helloworld` POD, in the namespace `default`, generates the following log entry:

```text
[2023-05-15T12:48:31.605Z] "- - -" 0 UH - - "-" 0 0 0 - "-" "-" "-" "-" "-" BlackHoleCluster - 10.108.186.1:8080 172.17.247.52:53742 - -
```

`BlackHoleCluster` resembles the same behavior as on the section [Curl helloworld.foo.svc.cluster.local:8080](#curl-helloworldfoosvcclusterlocal--8080).







### from `helloworld-foo`

Reminder of the **egress** criteria that has been configured to be met:

[ ] Port `8080`.

[ ] `HTTP` protocol.

[ ] Namespace destination `foo` or `default`.




#### Curl helloworld.foo.svc.cluster.local:80

On this scenario we meet the following criteria:

[ ] Port `8080`.

[x] `HTTP` protocol.

[x] Namespace destination `foo` or `default`.

```shell
NAMESPACE="foo" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.foo.svc.cluster.local:80 -sI
```

```text
command terminated with exit code 56
```


##### What's happening?

Let's observe the logs activity from the `istio-proxy` container, of the deployment `helloworld` in the namespace `foo` when we send request towards the service `helloworld` in the namespace `foo` through the port `80`.

```shell
NAMESPACE="foo" && kubectl logs -l app=helloworld --follow -c istio-proxy -n $NAMESPACE --tail 0
```

From another `shell` send a request towards the destination.

```shell
NAMESPACE="foo" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.foo.svc.cluster.local:80 -sI
```

We can see, how the `istio-proxy` container, from the `helloworld` POD, in the namespace `foo`, generates the following log entry:

```text
[2023-05-15T12:56:49.064Z] "- - -" 0 UH - - "-" 0 0 0 - "-" "-" "-" "-" "-" BlackHoleCluster - 10.107.249.242:80 172.17.121.93:57680 - -
```

`BlackHoleCluster` resembles the same behavior as on the section [Curl helloworld.foo.svc.cluster.local:8080](#curl-helloworldfoosvcclusterlocal--8080).






#### Curl helloworld.foo.svc.cluster.local:8080

On this scenario we meet the following criteria:

[x] Port `8080`.

[x] `HTTP` protocol.

[x] Namespace destination `foo` or `default`.

```shell
NAMESPACE="foo" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.foo.svc.cluster.local:8080 -sI
```

```text
HTTP/1.1 200 OK
server: envoy
date: Mon, 15 May 2023 12:57:58 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 77
```





#### Curl helloworld.default.svc.cluster.local:80

On this scenario we meet the following criteria:

[ ] Port `8080`.

[x] `HTTP` protocol.

[x] Namespace destination `foo` or `default`.

```shell
NAMESPACE="foo" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.default.svc.cluster.local:80 -sI
```

```text
command terminated with exit code 56
```

##### What's happening?

Let's observe the logs activity from the `istio-proxy` container, of the deployment `helloworld` in the namespace `foo` when we send request towards the service `helloworld` in the namespace `default` through the port `80`.

```shell
NAMESPACE="foo" && kubectl logs -l app=helloworld --follow -c istio-proxy -n $NAMESPACE --tail 0
```

From another `shell` send a request towards the destination.

```shell
NAMESPACE="foo" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.default.svc.cluster.local:80 -sI
```

We can see, how the `istio-proxy` container, from the `helloworld` POD, in the namespace `foo`, generates the following log entry:

```text
[2023-05-15T13:03:50.935Z] "- - -" 0 UH - - "-" 0 0 0 - "-" "-" "-" "-" "-" BlackHoleCluster - 10.108.186.1:80 172.17.121.93:43342 - -
```

`BlackHoleCluster` resembles the same behavior as on the section [Curl helloworld.foo.svc.cluster.local:8080](#curl-helloworldfoosvcclusterlocal--8080).





#### Curl helloworld.default.svc.cluster.local:8080

On this scenario we meet the following criteria:

[x] Port `8080`.

[x] `HTTP` protocol.

[x] Namespace destination `foo` or `default`.

```shell
NAMESPACE="foo" && kubectl exec -n ${NAMESPACE} "$(kubectl get pod -n ${NAMESPACE} -l app=helloworld -o jsonpath={.items..metadata.name})" -- curl helloworld.default.svc.cluster.local:8080 -sI
```

```text
HTTP/1.1 200 OK
server: envoy
date: Mon, 15 May 2023 13:07:49 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 67
```

## BlackHoleCluster?

Let's check the learned routes from each deployment.

### helloworld-default

```shell
NAMESPACE="default" && istioctl proxy-config clusters -n $NAMESPACE "$(kubectl get pods -n ${NAMESPACE} -l app=helloworld | tail -n 1 | awk '{ print $1 }')"
```
```text
SERVICE FQDN                         PORT     SUBSET     DIRECTION     TYPE             DESTINATION RULE
                                     80       -          inbound       ORIGINAL_DST     
BlackHoleCluster                     -        -          -             STATIC           
InboundPassthroughClusterIpv4        -        -          -             ORIGINAL_DST     
PassthroughCluster                   -        -          -             ORIGINAL_DST     
agent                                -        -          -             STATIC           
helloworld.foo.svc.cluster.local     80       -          outbound      EDS              
prometheus_stats                     -        -          -             STATIC           
sds-grpc                             -        -          -             STATIC           
xds-grpc                             -        -          -             STATIC           
zipkin                               -        -          -             STRICT_DNS
```

We can observe the following entries:

- `BlackHoleCluster                     -        -          -             STATIC`

and

- `helloworld.foo.svc.cluster.local     80       -          outbound      EDS`

Where `BlackHoleCluster` is a static destination without port attributed nor direction set, and is the route used to send the traffic to the `void`.

As well, we can find the route `helloworld.foo.svc.cluster.local` that specifies the port `80` and direction `outbound`.

> **Note:**\
> For more information about the routes, refer to the [documentation about `pilot-discovery`](https://istio.io/latest/docs/reference/commands/pilot-discovery/#pilot-discovery-completion).


### helloworld-foo

```shell
NAMESPACE="foo" && istioctl proxy-config clusters -n $NAMESPACE "$(kubectl get pods -n ${NAMESPACE} -l app=helloworld | tail -n 1 | awk '{ print $1 }')"
```
```text
SERVICE FQDN                             PORT     SUBSET     DIRECTION     TYPE             DESTINATION RULE
                                         80       -          inbound       ORIGINAL_DST     
BlackHoleCluster                         -        -          -             STATIC           
InboundPassthroughClusterIpv4            -        -          -             ORIGINAL_DST     
PassthroughCluster                       -        -          -             ORIGINAL_DST     
agent                                    -        -          -             STATIC           
helloworld.default.svc.cluster.local     8080     -          outbound      EDS              
helloworld.foo.svc.cluster.local         8080     -          outbound      EDS              
prometheus_stats                         -        -          -             STATIC           
sds-grpc                                 -        -          -             STATIC           
xds-grpc                                 -        -          -             STATIC           
zipkin                                   -        -          -             STRICT_DNS
```

We can observe the following entries:

- `BlackHoleCluster                     -        -          -             STATIC`

and

- `helloworld.foo.svc.cluster.local     80       -          outbound      EDS`

Where `BlackHoleCluster` is a static destination without port attributed nor direction set, and is the route used to send the traffic to the `void`.

As well, we can find the routes `helloworld.foo.svc.cluster.local` and `helloworld.default.svc.cluster.local` where both specify the port `8080` and direction `outbound`.

> **Note:**\
> For more information about the routes, refer to the [documentation about `pilot-discovery`](https://istio.io/latest/docs/reference/commands/pilot-discovery/#pilot-discovery-completion).


## Cleanup

Finally, a cleanup from the resources deployed.

```shell
kubectl delete -f ./
```
```text
namespace "foo" deleted
deployment.apps "helloworld-default" deleted
deployment.apps "helloworld-foo" deleted
service "helloworld" deleted
service "helloworld" deleted
sidecar.networking.istio.io "helloworld-sidecar-default" deleted
sidecar.networking.istio.io "helloworld-sidecar-foo" deleted
```


# Links of interest

- https://istio.io/latest/docs/reference/config/networking/sidecar/#IstioEgressListener

- https://istio.io/latest/blog/2019/monitoring-external-service-traffic/#what-are-blackhole-and-passthrough-clusters

- https://istio.io/v1.0/help/ops/traffic-management/proxy-cmd/#deep-dive-into-envoy-configuration

- https://istio.io/latest/docs/reference/commands/pilot-discovery/#pilot-discovery-completion