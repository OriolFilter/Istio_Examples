---
gitea: none
include_toc: true
---


## Description

This example displays how to configure a `circuit breaking` in Istio, as well on how to test it and trigger the limits sets.

## Based on

This example is based on the [ **OFFICIAL** Istio documentation example regarding circuit breaking](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/).

## Fortio

aka. `Fortio` allows you to send traffic requests meanwhile being allowed to have some degree of control over them.

This is useful as we will try to reach/trigger the limits set in the DestinationRule configuration.

# Configuration

## Service

The service will forward incoming traffic from the service port `8080`, that will be forwarded towards the port `80` from the deployment, which contains an `HTTP` service.


```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
    service: helloworld
spec:
  ports:
    - port: 8080
      name: http
      targetPort: 80
      protocol: TCP
      appProtocol: http
  selector:
    app: helloworld
```

## Deployment

The deployment listen to the port `80` and `443`, hosting an `HTTP` and `HTTPS` service respectively to the aforementioned ports.

> **Note:**\
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-nginx-demo)


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld
  labels:
    app: helloworld
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
          image: oriolfilter/https-nginx-demo
          resources:
            requests:
              cpu: "100m"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
            - containerPort: 443
```

## Destination rule

This destination rule configures and sets limits to the traffic with host destination `helloworld.default.svc.cluster.local`.

- `connectionPool.tcp.maxConnections` being set to 1, limits the amount of simultaneous maximum number of connections to 1.

- `connectionPool.http.http1MaxPendingRequests`: Number of queued requests.

- `connectionPool.http.maxRequestsPerConnection`: Limits the amount of connections to the backend by source of the request, if 1 is set in this field (which is our scenario), it disables the keep alive configuration.
  
- `outlierDetection.consecutive5xxErrors`: Number of status codes `5XX` required before a host is ejected from the connection pool.

- `outlierDetection.interval`: Time between each analysis.

- `outlierDetection.baseEjectionTime`: Minimum of time that a host is ejected from the connection pool.

- `outlierDetection.maxEjectionPercent`: Maximum of hosts available to be ejected, as we set it to 100%, and as well we have only 1 deployment, whenever this rule is required to be triggered, it will allow the trigger to proceed to remove the host from the connection pool, finally resulting in all the hosts to be ejected.

> **Note:**/
> For more information regarding `DestinationRules` and their configuration fields, reffer to the following [official Istio documentation regarding `DestinationRules`](https://istio.io/latest/docs/reference/config/networking/destination-rule/).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld.default.svc.cluster.local
spec:
  host: helloworld.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
```


# Walkthrough

## Deploy resources

```shell
kubectl apply -f ./
```
```text
service/helloworld created
deployment.apps/helloworld created
service/fortio created
deployment.apps/fortio-deploy created
destinationrule.networking.istio.io/helloworld.default.svc.cluster.local created
```

## Test deployments
### helloworld.default.svc.cluster.local
#### Check connectivity from `fortio` to `helloworld` 

We will use the package `/usr/bin/fortio` to send a `curl` request towards the `helloworld` service deployment. 

If it doesn't work, ensure that the URL And IP are the right ones, as well if there isn't a `AuthorizationPolicy` that limits the traffic (also ensure that the deployments are ready `kubectl get deployments -w -n default -owide`).

```shell
kubectl exec -n default  "$(kubectl get pod -n default -l app=fortio -o jsonpath={.items..metadata.name})" -- /usr/bin/fortio curl -quiet helloworld.default.svc.cluster.local:8080
```
```text
HTTP/1.1 200 OK
server: envoy
date: Sat, 29 Apr 2023 04:12:37 GMT
content-type: text/html
content-length: 15
last-modified: Tue, 25 Apr 2023 00:47:17 GMT
etag: "64472315-f"
strict-transport-security: max-age=7200
accept-ranges: bytes
x-envoy-upstream-service-time: 100

<h2>Howdy</h2>
```

#### Perform a stress test of the resources deployed

Through the `Fortio` container, we will execute the following command `fortio load -c 2 -qps 0 -n 20`:

- `fortio`: The package used.
- `load`: It's used to gather statistics.
- `-c 2`: Number of simultaneous connections.
- `-qps 0`: Queries per second, if it's set to `0`, means that there is no limit and will try to send it as fast/maximum as possible.
- `-n 20`: Send 20 queries.

> **Note:**\
> For more information regarding the available possible command configurations, refer to the respective [Fortio documentation on Github](https://github.com/fortio/fortio#command-line-arguments)

```shell
kubectl exec -n default  "$(kubectl get pod -n default -l app=fortio -o jsonpath={.items..metadata.name})"  -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning helloworld.default.svc.cluster.local:8080
```

```text
04:16:28 I logger.go:183> Log level is now 3 Warning (was 2 Info)
Fortio 1.54.2 running at 0 queries per second, 8->8 procs, for 20 calls: helloworld.default.svc.cluster.local:8080
04:16:28 W http_client.go:170> Assuming http:// on missing scheme for 'helloworld.default.svc.cluster.local:8080'
Starting at max qps with 2 thread(s) [gomax 8] for exactly 20 calls (10 per thread + 0)
04:16:28 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
04:16:28 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
04:16:28 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
04:16:28 W http_client.go:1058> [1] Non ok http code 503 (HTTP/1.1 503)
04:16:28 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
04:16:28 W http_client.go:1058> [1] Non ok http code 503 (HTTP/1.1 503)
04:16:28 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
04:16:28 W http_client.go:1058> [1] Non ok http code 503 (HTTP/1.1 503)
Ended after 259.87612ms : 20 calls. qps=76.96
Aggregated Function Time : count 20 avg 0.02460594 +/- 0.02677 min 0.0010981 max 0.090700113 sum 0.492118798
# range, mid point, percentile, count
>= 0.0010981 <= 0.002 , 0.00154905 , 15.00, 3
> 0.002 <= 0.003 , 0.0025 , 20.00, 1
> 0.003 <= 0.004 , 0.0035 , 25.00, 1
> 0.005 <= 0.006 , 0.0055 , 30.00, 1
> 0.007 <= 0.008 , 0.0075 , 40.00, 2
> 0.011 <= 0.012 , 0.0115 , 45.00, 1
> 0.012 <= 0.014 , 0.013 , 55.00, 2
> 0.016 <= 0.018 , 0.017 , 60.00, 1
> 0.018 <= 0.02 , 0.019 , 65.00, 1
> 0.025 <= 0.03 , 0.0275 , 75.00, 2
> 0.03 <= 0.035 , 0.0325 , 80.00, 1
> 0.05 <= 0.06 , 0.055 , 85.00, 1
> 0.07 <= 0.08 , 0.075 , 95.00, 2
> 0.09 <= 0.0907001 , 0.0903501 , 100.00, 1
# target 50% 0.013
# target 75% 0.03
# target 90% 0.075
# target 99% 0.0905601
# target 99.9% 0.0906861
Error cases : count 8 avg 0.012249498 +/- 0.01797 min 0.0010981 max 0.054279663 sum 0.097995986
# range, mid point, percentile, count
>= 0.0010981 <= 0.002 , 0.00154905 , 37.50, 3
> 0.002 <= 0.003 , 0.0025 , 50.00, 1
> 0.003 <= 0.004 , 0.0035 , 62.50, 1
> 0.005 <= 0.006 , 0.0055 , 75.00, 1
> 0.025 <= 0.03 , 0.0275 , 87.50, 1
> 0.05 <= 0.0542797 , 0.0521398 , 100.00, 1
# target 50% 0.003
# target 75% 0.006
# target 90% 0.0508559
# target 99% 0.0539373
# target 99.9% 0.0542454
# Socket and IP used for each connection:
[0]   6 socket used, resolved to 10.99.49.188:8080, connection timing : count 6 avg 0.00037803967 +/- 0.0001574 min 0.000231869 max 0.00069415 sum 0.002268238
[1]   4 socket used, resolved to 10.99.49.188:8080, connection timing : count 4 avg 0.00045579175 +/- 9.155e-05 min 0.0003847 max 0.000612777 sum 0.001823167
Connection time (s) : count 10 avg 0.0004091405 +/- 0.0001403 min 0.000231869 max 0.00069415 sum 0.004091405
Sockets used: 10 (for perfect keepalive, would be 2)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.99.49.188:8080: 10
Code 200 : 12 (60.0 %)
Code 503 : 8 (40.0 %)
Response Header Sizes : count 20 avg 167.7 +/- 136.9 min 0 max 280 sum 3354
Response Body/Total Sizes : count 20 avg 273.1 +/- 26.21 min 241 max 295 sum 5462
All done 20 calls (plus 0 warmup) 24.606 ms avg, 77.0 qps
```

#### Information to highlight

From the output received, I would like to focus in the following entry, which states that 60% of the traffic was successful (returning a status code `200`), meanwhile 40% failed (returning status code `503`).

```text
Code 200 : 12 (60.0 %)
Code 503 : 8 (40.0 %)
```

### Check Fortio istio-proxy logs (`pilot-agent request GET stats`)

Check (Fortio's app) istio-proxy logs

```shell
kubectl exec "$(kubectl get pod -n default -l app=fortio -o jsonpath={.items..metadata.name})" -c istio-proxy -- pilot-agent request GET stats | grep helloworld | grep pending
```
```text
cluster.outbound|8080||helloworld.default.svc.cluster.local.circuit_breakers.default.remaining_pending: 1
cluster.outbound|8080||helloworld.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|8080||helloworld.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|8080||helloworld.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8080||helloworld.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8080||helloworld.default.svc.cluster.local.upstream_rq_pending_overflow: 3
cluster.outbound|8080||helloworld.default.svc.cluster.local.upstream_rq_pending_total: 18
```

If we review the field `upstream_rq_pending_overflow`, where it states that the value is set to `3`, it means that 3 entries where flagged for circuit breaking.

## helloworld.default

### Test destination URL `helloworld.default`

Same procedure as the step [Perform a stress test of the resources deployed](#perform-a-stress-test-of-the-resources-deployed), but instead of using the destination URL `helloworld.default.svc.cluster.local`, we will be using the URL `helloworld.default` to confirm if the destination rule is still being applied even if the full URL doesn't match.

```shell
kubectl exec -n default  "$(kubectl get pod -n default -l app=fortio -o jsonpath={.items..metadata.name})"  -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning helloworld.default:8080
```

```text
20:01:10 I logger.go:183> Log level is now 3 Warning (was 2 Info)
Fortio 1.54.2 running at 0 queries per second, 8->8 procs, for 20 calls: helloworld.default:8080
20:01:10 W http_client.go:170> Assuming http:// on missing scheme for 'helloworld.default:8080'
Starting at max qps with 2 thread(s) [gomax 8] for exactly 20 calls (10 per thread + 0)
20:01:10 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [1] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [1] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [1] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [1] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [1] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [1] Non ok http code 503 (HTTP/1.1 503)
20:01:10 W http_client.go:1058> [0] Non ok http code 503 (HTTP/1.1 503)
Ended after 162.18683ms : 20 calls. qps=123.31
Aggregated Function Time : count 20 avg 0.014704745 +/- 0.01561 min 0.001096933 max 0.044131073 sum 0.294094897
# range, mid point, percentile, count
>= 0.00109693 <= 0.002 , 0.00154847 , 30.00, 6
> 0.003 <= 0.004 , 0.0035 , 35.00, 1
> 0.004 <= 0.005 , 0.0045 , 45.00, 2
> 0.005 <= 0.006 , 0.0055 , 50.00, 1
> 0.006 <= 0.007 , 0.0065 , 55.00, 1
> 0.011 <= 0.012 , 0.0115 , 60.00, 1
> 0.014 <= 0.016 , 0.015 , 65.00, 1
> 0.016 <= 0.018 , 0.017 , 70.00, 1
> 0.018 <= 0.02 , 0.019 , 75.00, 1
> 0.035 <= 0.04 , 0.0375 , 90.00, 3
> 0.04 <= 0.0441311 , 0.0420655 , 100.00, 2
# target 50% 0.006
# target 75% 0.02
# target 90% 0.04
# target 99% 0.043718
# target 99.9% 0.0440898
Error cases : count 11 avg 0.0029070851 +/- 0.001827 min 0.001096933 max 0.006039404 sum 0.031977936
# range, mid point, percentile, count
>= 0.00109693 <= 0.002 , 0.00154847 , 54.55, 6
> 0.003 <= 0.004 , 0.0035 , 63.64, 1
> 0.004 <= 0.005 , 0.0045 , 81.82, 2
> 0.005 <= 0.006 , 0.0055 , 90.91, 1
> 0.006 <= 0.0060394 , 0.0060197 , 100.00, 1
# target 50% 0.00190969
# target 75% 0.004625
# target 90% 0.0059
# target 99% 0.00603507
# target 99.9% 0.00603897
# Socket and IP used for each connection:
[0]   6 socket used, resolved to 10.98.152.137:8080, connection timing : count 6 avg 0.00047558633 +/- 0.0001364 min 0.000294869 max 0.000739941 sum 0.002853518
[1]   7 socket used, resolved to 10.98.152.137:8080, connection timing : count 7 avg 0.000457311 +/- 0.0001076 min 0.000320826 max 0.000596445 sum 0.003201177
Connection time (s) : count 13 avg 0.00046574577 +/- 0.0001221 min 0.000294869 max 0.000739941 sum 0.006054695
Sockets used: 13 (for perfect keepalive, would be 2)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.98.152.137:8080: 13
Code 200 : 9 (45.0 %)
Code 503 : 11 (55.0 %)
Response Header Sizes : count 20 avg 125.9 +/- 139.2 min 0 max 280 sum 2518
Response Body/Total Sizes : count 20 avg 265.2 +/- 26.76 min 241 max 295 sum 5304
All done 20 calls (plus 0 warmup) 14.705 ms avg, 123.3 qps
```

As we can see, the rules are still being applied, this time resulting in a 45% of the traffic receiving a successful status code (`200`), meanwhile a 55% received a failure status code (`503`).

This confirms that, even if the full URL is not the same as the configured in the [DestinationRule](#destination-rule), the DestinationRule is still being enforced.



## Cleanup

```shell
kubectl delete -f ./
```

```text
deployment.apps "helloworld" deleted
destinationrule.networking.istio.io "helloworld.default.svc.cluster.local" deleted
service "fortio" deleted
deployment.apps "fortio-deploy" deleted
service "helloworld" deleted
```

## Links of interest

- https://raw.githubusercontent.com/istio/istio/release-1.17/samples/httpbin/sample-client/fortio-deploy.yaml

- https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/#adding-a-client

- https://github.com/fortio/fortio

- https://github.com/fortio/fortio#command-line-arguments

- https://istio.io/latest/docs/reference/config/networking/destination-rule/

- https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-TCPSettings

- https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-HTTPSettings