---
gitea: none
include_toc: true
---


# Description

On this example we will be deploying an `AuthorizationPolicy` object to control the traffic that the `envoy-proxy` will manage on deployment created.

As well, we will configure the `AuthorizationPolicy` object will be applied to the deployments with the targeted through the usage of labels to filter the resources affected.

> **Note:**\
> On this example there is minimal changes to the configuration to involve targeting the deployment resources through label filtering.

# Based on

- [01-AuthorizationPolicy-Target-Namespaces](../01-AuthorizationPolicy-Target-Namespaces)

[//]: # (## Description)

[//]: # ()
[//]: # (Bla bla bla)

[//]: # ()
[//]: # (In this example we will be targeting the labels set to the deployments, while keeping part of the previous AuthorizationPolicy configuration to maintain its behavior.   )

[//]: # (For such, it's important to check the labels set in the Istio ingress that we will be using.)

[//]: # ()
[//]: # (On my case, in the gateway I will be targeting/using the Istio ingress `ingressgateway`.)

[//]: # ()
[//]: # (I would **strongly** recommend checking yours through the following command, as to proceed we should be aware of which are our possible labels options.)

[//]: # ()
[//]: # (```shell)

[//]: # (kubectl get deployments -A -l istio=ingressgateway -o jsonpath='{.items[].spec.template.metadata.labels}'| jq)

[//]: # (```)

[//]: # (```json)

[//]: # ({)

[//]: # (  "app": "istio-ingressgateway",)

[//]: # (  "chart": "gateways",)

[//]: # (  "heritage": "Tiller",)

[//]: # (  "install.operator.istio.io/owning-resource": "unknown",)

[//]: # (  "istio": "ingressgateway",)

[//]: # (  "istio.io/rev": "default",)

[//]: # (  "operator.istio.io/component": "IngressGateways",)

[//]: # (  "release": "istio",)

[//]: # (  "service.istio.io/canonical-name": "istio-ingressgateway",)

[//]: # (  "service.istio.io/canonical-revision": "latest",)

[//]: # (  "sidecar.istio.io/inject": "false")

[//]: # (})

[//]: # (```)

[//]: # ()
[//]: # (Based on the list displayed, I would suggest focusing on the following options:)

[//]: # ()
[//]: # (```json)

[//]: # ({)

[//]: # ("istio": "ingressgateway",)

[//]: # ("operator.istio.io/component": "IngressGateways",)

[//]: # ("app": "istio-ingressgateway",)

[//]: # (})

[//]: # (```)

[//]: # ()
[//]: # (The label `"service.istio.io/canonical-revision": "latest"` could be reasonable to use, in very specific situations, as depending on the implementation/environment or procedures that we might use in the future, it's something to keep in mind in case of being configured.)

[//]: # ()
[//]: # ()


# Configuration

## AuthorizationPolicy
### Allow nothing (deny all not matched)

#### default namespace

If the action is not specified, it will deploy the rule as "ALLOW".

Here we are deploying a rule that allows the traffic that it matches, yet as it has no conditions, it will never match.

```yaml
# Deny all requests to namespace default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-nothing
  namespace: default
```

Citing the [Authorization Policy documentation from Istio](https://istio.io/latest/docs/reference/config/security/authorization-policy), regarding the evaluation behavior of these rules:

    1. If there are any CUSTOM policies that match the request, evaluate and deny the request if the evaluation result is deny.
    2. If there are any DENY policies that match the request, deny the request.
    3. If there are no ALLOW policies for the workload, allow the request.
    4. If any of the ALLOW policies match the request, allow the request.
    5. Deny the request.

On this scenario, as we don't have any DENY or CUSTOM rule, we skip right into the 3rd scenario.

This rule is being applied to the workload (due being a rule that affects the whole namespace), and for such the 3rd scenario is not being applied either.

On the 4rth, scenario, as the rule deployed, even if it's on ALLOW mode, has no conditions, it won't allow the traffic either.

And finally, as any of the above scenarios allowed the traffic of the request, it ends getting denied.

For such, the creation of this "empty" rule, has set the authorization mode on the not explicitly allowed request to "DENY ALL".

#### foo namespace

Same behavior as above, this time applied to the namespace `foo`

```yaml
# Deny all requests to namespace foo
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-nothing
  namespace: foo
spec:
  {}
```

## ALLOW


#### byeworld-allow-from-istio-system

As we have a service deployed, and the traffic will come through the Istio Load Balancer (at least on my environment). I have set a rule that will allow all the traffic coming from a resource located in the namespace `istio-system`.

This rule will be applied to the deployments that have set the following label `app: byeworld`, and deployed in the namespace `istio-system`.

> **Note:**\
> As this rule will be deployed in the root namespace `istio-system` (it's my root namespace in **MY** environment, review your Istio configuration to ensure which is **YOUR** root namespace).\
> By deploying the rule in the root namespace, it gets applied to all namespaces, I have set this to ensure that there are minor differences on the configuration in comparison on which example this is based on. As well this will allow us to confirm tha the labels are being applied correctly.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: byeworld-allow-from-istio-system
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: byeworld
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: ["istio-system"]
```

#### byeworld-allow-head-from-default

I have set a new rule, that will allow the traffic coming from the namespace `default`, as long the method used is `HEAD` and is not targeting the path `/secret`.

This rule will be applied to the deployments that have set the following label `app: byeworld`, and deployed in the namespace `istio-system`.

> **Note:**\
> This will be deployed in the root namespace `istio-system` (it's my root namespace in **MY** environment, review your Istio configuration to ensure which is **YOUR** root namespace).\
> By deploying the rule in the root namespace, it gets applied to all namespaces, I have set this to ensure that there are minor differences on the configuration in comparison on which example this is based on. As well this will allow us to confirm tha the labels are being applied correctly.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: byeworld-allow-head-from-default
  namespace: istio-system
spec:
  action: ALLOW
  selector:
    matchLabels:
      app: byeworld
  rules:
    - from:
        - source:
            namespaces: ["default"]
      to:
        - operation:
            methods: ["HEAD"]
            notPaths: ["/secret*"]
```


# Walkthrough

## Deploy the resources

```shell
kubectl apply -f ./
```
```text
namespace/foo created
authorizationpolicy.security.istio.io/allow-nothing created
authorizationpolicy.security.istio.io/byeworld-allow-from-istio-system created
authorizationpolicy.security.istio.io/byeworld-allow-head-from-default created
service/helloworld created
deployment.apps/helloworld-nginx created
service/byeworld created
deployment.apps/byeworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs create
```

## Test resources

### Curl / LB requests / requests from external traffic

#### Get LB IP

```shell
kubectl get svc istio-ingressgateway -n istio-system 
```
```text
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```

#### helloworld

Due to the rule `allow-nothing` created on the namespace `istio-system`, which is being applied to all the namespaces, we are not hitting any rule that explicitly allows us, and for such, the traffic is being denied.

For such we receive the status code `403` (**Forbidden**)

```shell
curl 192.168.1.50/helloworld -I    
```
```text
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Thu, 27 Apr 2023 01:20:06 GMT
server: istio-envoy
x-envoy-upstream-service-time: 108
```

#### byeworld

As we created the rule `byeworld-allow-from-istio-system` created in the namespace `foo`, which allows all the traffic coming from a resource located in the namespace `istio-system`, and the load balancer used is located in the namespace `istio-system`, the traffic is allowed.

For such we receive the code `200`.

```shell
curl 192.168.1.50/byeworld --head    
```
```text
HTTP/1.1 200 OK
server: istio-envoy
date: Thu, 27 Apr 2023 01:20:49 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 104
```

### Connectivity between the deployments

> **NOTE:**\
> The command `curl`, when uses the flag `--head` or `-I`, the request sent will be a `HEAD` request.
>
> It's important to be aware of that due the rule configured, where one of the targets was the method used, specifically targeted the method `HEAD`.
>
> On this example, all request will be done with the method `HEAD` unless specified otherwise.

#### helloworld towards byeworld

It works.

Due to the rule `byeworld-allow-head-from-default` deployed on the namespace `foo`, which allowed the traffic coming from the namespace `default` as long it used the method `HEAD` and wasn't targeting the path `/secret`, the request is allowed.


```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl http://byeworld.foo.svc.cluster.local:9090 --head
```
```text
HTTP/1.1 200 OK
server: envoy
date: Thu, 27 Apr 2023 01:20:58 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 86
```

#### helloworld towards byeworld (GET REQUEST)

This example is made on base on the last comand executed, where the request sent uses the `HEAD` method.

On this example the flag `--head` is removed, which causes the command `curl` to send a request of method `GET`.

As the rule created required the method to be `HEAD`, it causes the request to not be allowed, and finally as there are no rules that allow this request, it results in failure.

```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl http://byeworld.foo.svc.cluster.local:9090  
```
```text
RBAC: access denied%
```

#### byeworld towards helloworld

It fails.

As expected, like when accessing through the Load Balancer, we receive the status code `403` (**Forbidden**).

The `HEAD` request is irrelevant on this scenario, yet using it as I like this output more.

```shell
kubectl exec -i -n foo -t "$(kubectl get pod -n foo -l app=byeworld | tail -n 1 | awk '{print $1}')" -- curl http://helloworld.default.svc.cluster.local:8080 --head
```
```text
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Thu, 27 Apr 2023 01:21:10 GMT
server: envoy
x-envoy-upstream-service-time: 96
```

#### helloworld towards byeworld/secret

Due to the configuration set on the rule `byeworld-allow-head-from-default`, one of the conditions for it to allow the traffic, was to not access the path/match the prefix expression  `/secret*`.

This causes the traffic to not be allowed.

```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl http://byeworld.foo.svc.cluster.local:9090/secret --head
```
```text
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Thu, 27 Apr 2023 01:21:18 GMT
server: envoy
x-envoy-upstream-service-time: 3
```


#### helloworld towards byeworld/not-found

On this example, we can notice how even if the request was allowed due meeting all the requirements, it still results in the error code `404` (Not Found).

This 404 error is raised by the destination service, yet before being able to handle such request, firstly the traffic required to be allowed, meaning that even if we target as a destination path a non-existent resource, we will need to match the requirements for the traffic to be allowed.

```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl http://byeworld.foo.svc.cluster.local:9090/not-found --head
```
```text
HTTP/1.1 404 Not Found
server: envoy
date: Thu, 27 Apr 2023 01:21:31 GMT
content-type: text/html
content-length: 153
x-envoy-upstream-service-time: 56
```

# Links of interest

- https://istio.io/latest/docs/reference/config/security/authorization-policy/