

# Default format

# Following traces

> Need to confirm this.

The default logs format incorporate a field for the trace ID

```shell
kubectl logs -n istio-ingress istio-ingress-68774dc847-dxw97 | grep 020e467a-c234-4a8f-b9f4-ca156ca06578
```

```text
[2025-07-22T17:55:52.084Z] "GET / HTTP/1.1" 403 - via_upstream - "-" 0 19 0 0 "10.244.1.1" "curl/8.14.1" "020e467a-c234-4a8f-b9f4-ca156ca06578" "172.18.10.10" "10.244.1.9:80" outbound|80||nginx-svc.default.svc.cluster.local 10.244.1.6:35472 10.244.1.6:80 10.244.1.1:14893 - -
```

On this log the trace ID is `020e467a-c234-4a8f-b9f4-ca156ca06578`, returned from the endpoint `nginx-svc.default.svc.cluster.local`, specifically the pod with IP `10.244.1.9`.

```shell
kubectl get pods -n default -owide | grep 10.244.1.9   
```

```text
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
nginx-deployment-6f595bd9d7-5jkns   2/2     Running   0          10m   10.244.1.9   istio-testing-worker   <none>           <none>
```

If we wanted to know why the status code 403 was returned, we would need to check for the respective log of traffic request, thus we can filter in the istio-proxy container by the trace ID.

```shell
kubectl logs nginx-deployment-6f595bd9d7-5jkns -c istio-proxy | grep 020e467a-c234-4a8f-b9f4-ca156ca06578
```

```text
[2025-07-22T17:55:52.085Z] "GET / HTTP/1.1" 403 - rbac_access_denied_matched_policy[none] - "-" 0 19 0 - "10.244.1.1" "curl/8.14.1" "020e467a-c234-4a8f-b9f4-ca156ca06578" "172.18.10.10" "-" inbound|80|| - 10.244.1.9:80 10.244.1.1:0 outbound_.80_._.nginx-svc.default.svc.cluster.local default
```

On this scenario it was caused by an "access denied" from the AuthorizationPolicy.