---
gitea: none
include_toc: true
---

# Istioctl analyze

`istioctl analyze` reviews the current configuration set.

Can be helpful to spot some improvements on the current configurations set, as well of the possibility of displaying misconfigurations / lack of them that might be causing issues.

```shell
istioctl analyze
```
```text
✔ No validation issues found when analyzing namespace: default.
```

By using the flag -A, it will review from all namespaces

```shell
istioctl analyze -A
```
```text
Info [IST0102] (Namespace istio-operator) The namespace is not enabled for Istio injection. Run 'kubectl label namespace istio-operator istio-injection=enabled' to enable it, or 'kubectl label namespace istio-operator istio-injection=disabled' to explicitly mark it as not needing injection.
Info [IST0118] (Service istio-system/grafana) Port name service (port: 3000, targetPort: 3000) doesn't follow the naming convention of Istio port.
Info [IST0118] (Service istio-system/jaeger-collector) Port name jaeger-collector-grpc (port: 14250, targetPort: 14250) doesn't follow the naming convention of Istio port.
Info [IST0118] (Service istio-system/jaeger-collector) Port name jaeger-collector-http (port: 14268, targetPort: 14268) doesn't follow the naming convention of Istio port.
```

One can specify/target a single namespace by using the flag `-n`

```shell
istioctl analyze -n istio-operator
```
```text
Info [IST0102] (Namespace istio-operator) The namespace is not enabled for Istio injection. Run 'kubectl label namespace istio-operator istio-injection=enabled' to enable it, or 'kubectl label namespace istio-operator istio-injection=disabled' to explicitly mark it as not needing injection.
```

## Example of spotting a misconfiguration

In this example, I have configured the gateway to listen to a port that currently is not open in the Isito Load Balancer selected.

```shell
istioctl analyze
```
```text
Warning [IST0104] (Gateway default/helloworld-gateway) The gateway refers to a port that is not exposed on the workload (pod selector istio=ingressgateway; port 81)
```

# Start the packet capture process on the istio-proxy container from a pod.

Target a pod and start a packet capture on the istio-proxy container.

This step requires Istio to be installed with the flag `values.global.proxy.privileged=true`

This is very useful to confirm if the service is receiving any traffic, or which is the traffic received.

If mTLS is  enabled and configured, the traffic received should be encrypted.

```shell
kubectl exec -n default  "$(kubectl get pod -n default -l app=helloworld -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port 80  -A
```
```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
...
```

# Logs

> **Note:**\
> Remember that you can use the command `watch` or `watch -n 5` (where 5 refers every 5 seconds) in case of being interested on execute this commands periodically.

## Istiod

```shell
kubectl logs -n istio-system -f deployments/istiod
```

## Istio-Proxy Pod

This will display the logs from a deployment while targeting the `istio-proxy` container from the targeted pod/deployment.

As well will attach the session to stream new logs. (`-f` `--follow`)

```shell
kubectl logs deployments/helloworld-default -f -c istio-proxy
```

```text
[2023-05-15T00:42:03.699Z] "- - -" 0 UH - - "-" 0 0 0 - "-" "-" "-" "-" "-" BlackHoleCluster - 10.111.90.232:8080 172.17.121.65:52006 - -
[2023-05-15T00:42:24.785Z] "HEAD / HTTP/1.1" 200 - via_upstream - "-" 0 0 2 1 "-" "curl/7.74.0" "c133cbf0-b57d-4fba-8f84-d683ab903399" "helloworld.default.svc.cluster.local" "172.17.121.65:80" inbound|80|| 127.0.0.6:51695 172.17.121.65:80 172.17.121.65:43786 outbound_.80_._.helloworld.default.svc.cluster.local default
[2023-05-15T00:42:24.784Z] "HEAD / HTTP/1.1" 200 - via_upstream - "-" 0 0 5 4 "-" "curl/7.74.0" "c133cbf0-b57d-4fba-8f84-d683ab903399" "helloworld.default.svc.cluster.local" "172.17.121.65:80" outbound|80||helloworld.default.svc.cluster.local 172.17.121.65:43786 10.111.90.232:80 172.17.121.65:57030 - default
[2023-05-15T00:43:23.209Z] "HEAD / HTTP/1.1" 200 - via_upstream - "-" 0 0 6 5 "-" "curl/7.74.0" "e1f0a2f3-93ff-4c41-8cb3-6d3a53fce065" "helloworld.foo.svc.cluster.local" "172.17.247.42:80" outbound|80||helloworld.foo.svc.cluster.local 172.17.121.65:55040 10.109.248.148:80 172.17.121.65:60520 - default
[2023-05-15T00:43:29.751Z] "- - -" 0 UH - - "-" 0 0 0 - "-" "-" "-" "-" "-" BlackHoleCluster - 10.109.248.148:8080 172.17.121.65:40370 - -
[2023-05-15T00:43:31.979Z] "- - -" 0 UH - - "-" 0 0 0 - "-" "-" "-" "-" "-" BlackHoleCluster - 10.109.248.148:8080 172.17.121.65:40402 - -
```


## Ingress

The service targeted, `istio-ingressgateway`, is an Ingress Load Balancer service from Istio.

```shell
kubectl logs -n istio-system services/istio-ingressgateway
```
#### Invalid TLS context has neither subject CN nor SAN names

The TLS certificate specified don't have the field CN or the field SAN.

To address this issue, issue a new certificate that has at least one of those fields.

#### initial fetch timed out for type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.Secretthread 

This is due not being able to retrieve the TLS configuration assigned to the gateway.

It's Important that the secret is located in the same namespace as the Istio Load Balancer used. In my case is the `istio-system`, but it will vary based on the environment.

# Istioctl proxy-config

## Check listeners

Useful to review which is the configuration assigned to an Istio ingress. / Confirm if the configuration we are intending to deploy is being applied / learned.

### Get Istio ingress pod name

> **Note:**\
> Depending on the ingress gateway set, and your environment, it could be that the Load Balancer is not located in the namespace `istio-system`.

```shell
kubectl get pods -n istio-system               
```

```text
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-6cb5b7fbb8-2nlp6                1/1     Running   0          2d3h
istio-ingressgateway-864db96c47-nvjc7   1/1     Running   0          20h
istiod-649d466b9-bwx7j                  1/1     Running   0          2d8h
jaeger-cc4688b98-h52xt                  1/1     Running   0          2d3h
kiali-594965b98c-zc67p                  1/1     Running   0          2d3h
prometheus-67f6764db9-szd5b             2/2     Running   0          2d3h

```

### List listeners

```shell
kubectl get pods -n istio-system istio-ingressgateway-864db96c47-nvjc7
```

```text
istioctl proxy-config listeners -n istio-system istio-ingressgateway-864db96c47-nvjc7
ADDRESS PORT  MATCH       DESTINATION
0.0.0.0 8443  SNI: lb.net Route: https.443.secure-http.helloworld-gateway.default
0.0.0.0 15021 ALL         Inline Route: /healthz/ready*
0.0.0.0 15090 ALL         Inline Route: /stats/prometheus*
```

This makes reference to the configuration set in the gateway resources.
Here we can notice a route with SNI match "lb.net", which is listening to the port 443 and HTTPS protocol.

## Check logs verbosity level settings

`istioctl proxy-config log` will display the verbosity level set from each log type for the specified pod.

```shell
istioctl proxy-config log helloworld-nginx-5d99f88767-cwcmd 
```
```text
helloworld-nginx-5d99f88767-cwcmd.default:
active loggers:
  admin: warning
  alternate_protocols_cache: warning
  aws: warning
  assert: warning
  backtrace: warning
  cache_filter: warning
  client: warning
  config: warning
  connection: warning
...
```

## List all

It displays ALL from the specified pod.

```shell
istioctl proxy-config all helloworld-nginx-5d99f88767-cwcmd
```
```txt
SERVICE FQDN                                               PORT      SUBSET     DIRECTION     TYPE             DESTINATION RULE
                                                           80        -          inbound       ORIGINAL_DST     
BlackHoleCluster                                           -         -          -             STATIC           
InboundPassthroughClusterIpv4                              -         -          -             ORIGINAL_DST     
PassthroughCluster                                         -         -          -             ORIGINAL_DST     
agent                                                      -         -          -             STATIC           
...
```

# Other links

## [Debugging with Istio](https://www.istioworkshop.io/12-debugging/01-istioctl-debug-command/)
