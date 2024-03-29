---
gitea: none
include_toc: true
---


# Description

On this example we will be deploying an `AuthorizationPolicy` object to control the traffic that the `envoy-proxy` will manage on deployment created.

As well, we will configure the `AuthorizationPolicy` object to be applied at a "namespace" level.

# Based on

- [10-mTLS_PeerAuthentication/01-mTLS](../../10-mTLS_PeerAuthentication/01-mTLS)

# Configuration

## Authentication configuration deployed

### default namespace

#### Allow nothing

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

### foo namespace

#### Allow nothing

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


#### allow-from-istio-system

As we have a service deployed, and the traffic will come through the Istio Load Balancer (at least on my environment). I have set a rule that will allow all the traffic coming from a resource located in the namespace `istio-system`.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-from-istio-system
  namespace: foo
spec:
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: ["istio-system"]
```

#### allow-get-from-default

As an additional example, I have set a new rule, that will allow the traffic coming from the namespace `default`, as long the method used is `HEAD` and is not targeting the path `/secret`. 

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-get-from-default
  namespace: foo
spec:
  action: ALLOW
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
authorizationpolicy.security.istio.io/allow-nothing created
authorizationpolicy.security.istio.io/allow-from-istio-system created
authorizationpolicy.security.istio.io/allow-head-from-default created
service/helloworld created
deployment.apps/helloworld-nginx created
service/byeworld created
deployment.apps/byeworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
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

Due to the rule `allow-nothing` created on the namespace `default`, we are not hitting any rule that explicitly allows us, and for such, the traffic is being denied.

For such we receive the status code `403` (**Forbidden**)

```shell
curl 192.168.1.50/helloworld -I    
```
```text
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Sat, 22 Apr 2023 02:00:34 GMT
server: istio-envoy
x-envoy-upstream-service-time: 11
```

#### byeworld

As we created the rule `allow-from-istio-system` created in the namespace `foo`, which allows all the traffic coming from a resource located in the namespace `istio-system`, and the load balancer used is located in the namespace `istio-system`, the traffic is allowed.

For such we receive the code `200`.

```shell
curl 192.168.1.50/byeworld --head    
```
```text
HTTP/1.1 200 OK
server: istio-envoy
date: Sat, 22 Apr 2023 02:01:48 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 91
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

Due to the rule `allow-head-from-default` deployed on the namespace `foo`, which allowed the traffic coming from the namespace `default` as long it used the method `HEAD` and wasn't targeting the path `/secret`, the request is allowed.



```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl http://byeworld.foo.svc.cluster.local:9090 --head
```
```text
HTTP/1.1 200 OK
server: envoy
date: Sat, 22 Apr 2023 02:08:56 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 6
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
date: Sat, 22 Apr 2023 02:06:21 GMT
server: envoy
x-envoy-upstream-service-time: 65
```

#### helloworld towards byeworld/secret

Due to the configuration set on the rule `allow-head-from-default`, one of the conditions for it to allow the traffic, was to not access the path/match the prefix expression  `/secret*`.

This causes the traffic to not be allowed.

```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl http://byeworld.foo.svc.cluster.local:9090/secret --head
```
```text
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Sat, 22 Apr 2023 02:40:30 GMT
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
date: Sat, 22 Apr 2023 06:09:59 GMT
content-type: text/html
content-length: 153
x-envoy-upstream-service-time: 67
```

# Links of interest

- https://istio.io/latest/docs/reference/config/security/authorization-policy/