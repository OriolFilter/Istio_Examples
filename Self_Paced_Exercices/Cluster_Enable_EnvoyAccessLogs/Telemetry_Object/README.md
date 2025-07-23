

Enable access logs for the whole "cluster level" making usage of the Telemetry object.


When checking the pod's istio-proxy we don't see access logs every time we send a request.

```shell
kubectl logs helloworld-v2-6ff4d84b5c-tlbcl -c istio-proxy -f
```

Once the envoy logs are enabled through Telemetry the requests will be logged.

```text
[2025-07-23T19:37:45.917Z] "HEAD / HTTP/1.1" 200 - via_upstream - "-" 0 0 0 0 "10.244.1.1" "curl/8.14.1" "4ece83a5-cfe9-4e4a-8971-2440053268ef" "172.18.10.10" "10.244.1.11:80" inbound|80|| 127.0.0.6:48301 10.244.1.11:80 10.244.1.1:0 outbound_.80_.default_.helloworld.default.svc.cluster.local default
[2025-07-23T19:37:46.294Z] "HEAD / HTTP/1.1" 200 - via_upstream - "-" 0 0 0 0 "10.244.1.1" "curl/8.14.1" "15f7393d-2f93-4f4e-bfb6-f5fcfc49d270" "172.18.10.10" "10.244.1.11:80" inbound|80|| 127.0.0.6:48301 10.244.1.11:80 10.244.1.1:0 outbound_.80_.default_.helloworld.default.svc.cluster.local default
[2025-07-23T19:37:47.110Z] "HEAD / HTTP/1.1" 200 - via_upstream - "-" 0 0 0 0 "10.244.1.1" "curl/8.14.1" "7911c90b-ab26-4f63-8c42-5158c7387e6c" "172.18.10.10" "10.244.1.11:80" inbound|80|| 127.0.0.6:48301 10.244.1.11:80 10.244.1.1:0 outbound_.80_.default_.helloworld.default.svc.cluster.local default
[2025-07-23T19:37:47.505Z] "HEAD / HTTP/1.1" 200 - via_upstream - "-" 0 0 0 0 "10.244.1.1" "curl/8.14.1" "8938a13f-b3e6-45e0-ba55-38f1b436b2d3" "172.18.10.10" "10.244.1.11:80" inbound|80|| 127.0.0.6:48301 10.244.1.11:80 10.244.1.1:0 outbound_.80_.default_.helloworld.default.svc.cluster.local default
[2025-07-23T19:37:48.303Z] "HEAD / HTTP/1.1" 200 - via_upstream - "-" 0 0 0 0 "10.244.1.1" "curl/8.14.1" "4a222d82-5e9a-458e-9502-d14d26581134" "172.18.10.10" "10.244.1.11:80" inbound|80|| 127.0.0.6:48301 10.244.1.11:80 10.244.1.1:0 outbound_.80_.default_.helloworld.default.svc.cluster.local default
[2025-07-23T19:37:48.668Z] "HEAD / HTTP/1.1" 200 - via_upstream - "-" 0 0 0 0 "10.244.1.1" "curl/8.14.1" "167c846d-836a-45c2-a002-646cdec6701a" "172.18.10.10" "10.244.1.11:80" inbound|80|| 127.0.0.6:48301 10.244.1.11:80 10.244.1.1:0 outbound_.80_.default_.helloworld.default.svc.cluster.local default
```


## Links

- https://istio.io/latest/docs/tasks/observability/logs/access-log/#enable-envoys-access-logging
- https://istio.io/latest/docs/tasks/observability/logs/telemetry-api/