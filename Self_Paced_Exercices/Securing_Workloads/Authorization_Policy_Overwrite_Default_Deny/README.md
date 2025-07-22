# Setup

Create an Authorization policy that allows traffic from **the nginx-ingress namespace** to **the port 80 of the Nginx backend**.

```shell
kubectl create -f ./src/
```

When doing a curl it will fail as follows:

```shell
curl 172.18.10.10
```

```shell
RBAC: access denied% 
```

In the logs it appears as `rbac_access_denied_matched_policy`.

```shell
kubectl logs nginx-deployment-6f595bd9d7-5jkns -c istio-proxy
```

```text
[2025-07-22T17:55:49.074Z] "GET / HTTP/1.1" 403 - rbac_access_denied_matched_policy[none] - "-" 0 19 0 - "10.244.1.1" "curl/8.14.1" "ff387ab3-186d-4e3f-9695-0f884acb4e20" "172.18.10.10" "-" inbound|80|| - 10.244.1.9:80 10.244.1.1:0 outbound_.80_._.nginx-svc.default.svc.cluster.local default
[2025-07-22T17:55:50.594Z] "GET / HTTP/1.1" 403 - rbac_access_denied_matched_policy[none] - "-" 0 19 0 - "10.244.1.1" "curl/8.14.1" "8b2193f2-746b-4ff1-851b-0b4bb4ef681f" "172.18.10.10" "-" inbound|80|| - 10.244.1.9:80 10.244.1.1:0 outbound_.80_._.nginx-svc.default.svc.cluster.local default
[2025-07-22T17:55:52.085Z] "GET / HTTP/1.1" 403 - rbac_access_denied_matched_policy[none] - "-" 0 19 0 - "10.244.1.1" "curl/8.14.1" "020e467a-c234-4a8f-b9f4-ca156ca06578" "172.18.10.10" "-" inbound|80|| - 10.244.1.9:80 10.244.1.1:0 outbound_.80_._.nginx-svc.default.svc.cluster.local default
```

After creating/applying the new AuthorizationPolicy to overwrite the existing behavior it should behave as follows:

```shell
curl 172.18.10.10 -s | grep h1
```

```shell
<h1>Welcome to nginx!</h1>
```


## Relevant links

- https://istio.io/latest/docs/reference/config/security/authorization-policy/

- https://istio.io/latest/docs/reference/config/security/conditions/

- https://istio.io/latest/docs/reference/config/security/normalization/