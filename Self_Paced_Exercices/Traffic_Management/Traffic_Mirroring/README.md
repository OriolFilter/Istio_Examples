
Mirror 100% of the traffic, so it goes both to the nginx(v1) and the apache(v2).

```shell
kubectl create -f ./src
```

```text
deployment.apps/helloworld-v1 created
deployment.apps/helloworld-v2 created
gateway.networking.istio.io/generic created
service/helloworld created
```

## Testing

As a good practice use the analyze command after applying resources, or beforehand analyze the yamls with -f

```shell
istioctl analyze ./
```

```text
✔ No validation issues found when analyzing:
  - DestinationRule.yaml
  - VirtualService.yaml
```


When sending a single curl request, it should be mirrored (and reflected) in both backends/pods, and the response received by the client will be from the Nginx backend (v1).


```shell
curl 172.18.10.10 | grep '<h1>'
```

```text
<h1>Welcome to nginx!</h1>
```

```shell
kubectl logs -f helloworld-v1-756bb9498c-wfqsr --tail=0
```

```text
127.0.0.6 - - [22/Jul/2025:19:58:40 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.14.1" "10.244.1.1"
127.0.0.6 - - [22/Jul/2025:19:58:41 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.14.1" "10.244.1.1"
127.0.0.6 - - [22/Jul/2025:19:58:42 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.14.1" "10.244.1.1"
```

v2

```shell
kubectl logs -f helloworld-v2-c79f8c86-55qr8 --tail=0
```

```text
127.0.0.6 - - [22/Jul/2025:19:58:40 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.14.1" "10.244.1.1"
127.0.0.6 - - [22/Jul/2025:19:58:41 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.14.1" "10.244.1.1"
127.0.0.6 - - [22/Jul/2025:19:58:42 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.14.1" "10.244.1.1"
```

If we check the Access Logs from the `istio-proxy` sidecar container.

```shell
kubectl logs helloworld-v1-756bb9498c-wfqsr -c istio-proxy | tail -n1
kubectl logs helloworld-v2-c79f8c86-55qr8 -c istio-proxy | tail -n1
```

v1

```text
[2025-07-22T20:02:39.527Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 615 0 0 "10.244.1.1" "curl/8.14.1" "8db9c925-02b9-43a9-a1a2-57d18951f4e4" "172.18.10.10" "10.244.1.15:80" inbound|80|| 127.0.0.6:34497 10.244.1.15:80 10.244.1.1:0 outbound_.80_.v1_.helloworld.default.svc.cluster.local default
```

v2

```text
[2025-07-22T20:02:39.528Z] "GET / HTTP/1.1" 400 - via_upstream - "-" 0 226 0 0 "10.244.1.1,10.244.1.6" "curl/8.14.1" "8db9c925-02b9-43a9-a1a2-57d18951f4e4" "172.18.10.10-shadow" "10.244.1.16:80" inbound|80|| 127.0.0.6:45225 10.244.1.16:80 10.244.1.6:0 outbound_.80_.v2_.helloworld.default.svc.cluster.local default
```

Besides the status code being 400 (note that in the logs the application reported a 200), meanwhile in the v1/original it stated "172.18.10.10", in the v2/mirrored it states "172.18.10.10-shadow".

## Links

- https://istio.io/latest/docs/tasks/traffic-management/

- https://istio.io/latest/docs/tasks/traffic-management/mirroring/

- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMirrorPolicy

