---
gitea: none
include_toc: true
---

# Continues from

[//]: # (- [01-hello_world_1_service_1_deployment]&#40;../../01-simple/01-hello_world_1_service_1_deployment&#41;)
- [01-namespaces](../01-namespaces)

> **Note:**\
> On this example there is minimal changes to the configuration to involve targeting service accounts.

## Description

Bla bla bla

Configuration targeting service accounts (among others)

By default, when a pod is deployed, if a service account has not been specified, it will be given the service account `default` from that namespace. 

# Changelog

## Service Account

### default namespace

#### istio-helloworld-sa

Created a service account named `istio-helloworld-sa`.

The label was set cause it made sense, yet it's not used on this example.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: istio-helloworld-sa
  labels:
    app: helloworld
```

## Authentication configuration deployed

### default namespace

#### Allow nothing

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

As we have a service deployed, and the traffic will come through the Istio Load Balancer (at least on my environment).

I have set a rule that will allow all the traffic coming from a resource located in the namespace `istio-system` AND also uses the service account `istio-ingressgateway-service-account` from that namespace.

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
        - source:
            principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
```

This service account is the account set to the ingress gateway resource currently set.

For reference, I have check it through the following command.

```shell
kubectl get pod -n istio-system istio-ingressgateway-864db96c47-mj5r2 -o jsonpath='{.spec.serviceAccount}'
```
```text
istio-ingressgateway-service-account%
```

#### allow-get-from-default

As an additional example, I have set a new rule, that will allow the traffic coming from the namespace `default`, as long the method used is `HEAD` and is not targeting the path `/secret`.\
Additionally, it requires that the requester uses the service account `istio-helloworld-sa` that we created.

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
            namespaces: ["default"]
        - source:
            principals: ["cluster.local/ns/default/sa/istio-helloworld-sa"]
      to:
        - operation:
            methods: ["HEAD"]
            notPaths: ["/secret*"]
```

Citing the [`rule.source.namespaces` field from the Authorization Policy documentation from Istio](https://istio.io/latest/docs/reference/config/security/authorization-policy/#Source):

> This field requires mTLS enabled and is the same as the source.namespace attribute.

# Walkthrough

## Deploy the resources

```shell
kubectl apply -f ./
```
```text
namespace/foo created
serviceaccount/istio-helloworld-sa created
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
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.173.231   192.168.1.50   15021:31277/TCP,80:30603/TCP,443:30290/TCP   34h
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
date: Sat, 22 Apr 2023 05:56:00 GMT
server: istio-envoy
x-envoy-upstream-service-time: 102
```

#### byeworld

We created the rule `allow-from-istio-system` created in the namespace `foo`, which allows all the traffic coming from a resource located in the namespace `istio-system`, and the load balancer used is located in the namespace `istio-system`.

On top of that, the Istio ingress being used, has the service account `istio-ingressgateway-service-account` from the namespace `istio-system` set, which is the current target of the rule.

For such we receive the code `200`.


```shell
curl 192.168.1.50/byeworld --head    
```
```text
HTTP/1.1 200 OK
server: istio-envoy
date: Sat, 22 Apr 2023 06:01:00 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 10
```

### Connectivity between the deployments

> **NOTE:**\
> The command `curl`, when uses the flag `--head` or `-I`, the request sent will be a `HEAD` request.
> 
> It's important to be aware of that due the rule configured, where one of the targets was the method used, specifically targeted the method `HEAD`.

#### helloworld towards byeworld (HEAD REQUEST)

It works.

Due to the rule `allow-get-from-default` deployed on the namespace `foo`, which allowed the traffic coming from the namespace `default` as long it used the method `HEAD` and wasn't targeting the path `/secret`, and, the deployment `helloworld` being using the service account `istio-helloworld-sa`, which is the target configured on the network rule, the request is allowed.

```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl http://byeworld.foo.svc.cluster.local:9090 --head
```
```text
HTTP/1.1 200 OK
server: envoy
date: Sat, 22 Apr 2023 06:01:08 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 4
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
date: Sat, 22 Apr 2023 06:06:13 GMT
server: envoy
x-envoy-upstream-service-time: 99
```

#### helloworld towards byeworld/secret

Due to the configuration set on the rule `allow-get-from-default`, one of the conditions for it to allow the traffic, was to not access the path/match the prefix expression  `/secret*`.

This causes the traffic to not be allowed.

```shell
kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl http://byeworld.foo.svc.cluster.local:9090/secret --head
```
```text
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Sat, 22 Apr 2023 06:15:38 GMT
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
date: Sat, 22 Apr 2023 06:15:29 GMT
content-type: text/html
content-length: 153
x-envoy-upstream-service-time: 28
```


# Links of interest

- https://istio.io/latest/docs/reference/config/security/authorization-policy/