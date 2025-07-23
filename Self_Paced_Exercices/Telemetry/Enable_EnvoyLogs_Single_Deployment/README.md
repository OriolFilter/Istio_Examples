

Enable the Envoy access logs only for the Apache deployment (v2).

Once the Telemetry object is enabled, only the istio-proxy container for the Apache backend should log the access entries from envoy.

```shell
kubectl logs -l backend=apache -f -c istio-proxy
```

```text
[2025-07-23T19:57:44.029Z] "HEAD /apache HTTP/1.1" 200 - via_upstream - "-" 0 0 0 0 "10.244.1.1" "curl/8.14.1" "5107e5b0-dda9-45f6-b419-e33695a3ca10" "172.18.10.10" "10.244.1.15:80" inbound|80|| 127.0.0.6:50565 10.244.1.15:80 10.244.1.1:0 outbound_.80_.apache_.helloworld.default.svc.cluster.local default
[2025-07-23T19:57:44.583Z] "HEAD /apache HTTP/1.1" 200 - via_upstream - "-" 0 0 0 0 "10.244.1.1" "curl/8.14.1" "e7fd3d4a-336c-4999-a7e7-7a8353063dd9" "172.18.10.10" "10.244.1.15:80" inbound|80|| 127.0.0.6:50565 10.244.1.15:80 10.244.1.1:0 outbound_.80_.apache_.helloworld.default.svc.cluster.local default
```


```shell
kubectl logs -l backend=nginx -f -c istio-proxy
```

```text

```