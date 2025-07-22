# Setup

Similar to the previous one, but this time enable traffic towards the paths except for `/admin*`, for both the `GET` and `HEAD` methods.



## GET

### Root

```shell
curl 172.18.10.10 -s | grep '<h1>'
```

```text
<h1>Welcome to nginx!</h1>
```

### /house

As expected since Nginx doesn't have that path we will receive a 404 **from Nginx**.

```shell
curl 172.18.10.10/house -s | grep '<h1>'
```

```text
<center><h1>404 Not Found</h1></center>
```

### /admin

Since the Regex matches the `notPath` rule, it won't be allowed to access.

```shell
curl 172.18.10.10/admin
curl 172.18.10.10/adminaa
curl 172.18.10.10/admin/123
curl 172.18.10.10/administrator/123    
```

```text
RBAC: access denied%
```

## HEAD

Same behavior than with GET

```shell
curl 172.18.10.10  -I
```

```text
HTTP/1.1 200 OK
server: istio-envoy
date: Tue, 22 Jul 2025 18:24:48 GMT
content-type: text/html
content-length: 612
last-modified: Tue, 04 Dec 2018 14:44:49 GMT
etag: "5c0692e1-264"
accept-ranges: bytes
x-envoy-upstream-service-time: 0
```

```shell
curl 172.18.10.10/admin/1234 -I
```

```text
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Tue, 22 Jul 2025 18:33:04 GMT
server: istio-envoy
x-envoy-upstream-service-time: 0
```

## Relevant links

- https://istio.io/latest/docs/reference/config/security/authorization-policy/

- https://istio.io/latest/docs/reference/config/security/conditions/

- https://istio.io/latest/docs/reference/config/security/normalization/