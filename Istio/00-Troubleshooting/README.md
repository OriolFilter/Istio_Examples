IDK put some text in there



### Start the packet capture process on the istio-proxy from a pod.

```shell
$ kubectl exec -n default  "$(kubectl get pod -n default -l app=helloworld -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port 80  -A
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

### Logs

Istio system logs

```shell
kubectl logs -f deployments/istiod -n istio-system
```



## Istioctl proxy-config


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
byeworld.foo.svc.cluster.local                             9090      -          outbound      EDS              
grafana.istio-system.svc.cluster.local                     3000      -          outbound      EDS              
helloworld.default.svc.cluster.local                       8080      -          outbound      EDS              
istio-ingressgateway.istio-system.svc.cluster.local        80        -          outbound      EDS              
istio-ingressgateway.istio-system.svc.cluster.local        443       -          outbound      EDS              
istio-ingressgateway.istio-system.svc.cluster.local        15021     -          outbound      EDS              
istiod.istio-system.svc.cluster.local                      443       -          outbound      EDS              
istiod.istio-system.svc.cluster.local                      15010     -          outbound      EDS              
istiod.istio-system.svc.cluster.local                      15012     -          outbound      EDS              
istiod.istio-system.svc.cluster.local                      15014     -          outbound      EDS              
jaeger-collector.istio-system.svc.cluster.local            9411      -          outbound      EDS              
jaeger-collector.istio-system.svc.cluster.local            14250     -          outbound      EDS              
jaeger-collector.istio-system.svc.cluster.local            14268     -          outbound      EDS              
kiali.istio-system.svc.cluster.local                       9090      -          outbound      EDS              
kiali.istio-system.svc.cluster.local                       20001     -          outbound      EDS              
kube-dns.kube-system.svc.cluster.local                     53        -          outbound      EDS              
kube-dns.kube-system.svc.cluster.local                     9153      -          outbound      EDS              
kubernetes.default.svc.cluster.local                       443       -          outbound      EDS              
myistio-ingressgateway.istio-ingress.svc.cluster.local     80        -          outbound      EDS              
myistio-ingressgateway.istio-ingress.svc.cluster.local     443       -          outbound      EDS              
myistio-ingressgateway.istio-ingress.svc.cluster.local     15021     -          outbound      EDS              
prometheus.istio-system.svc.cluster.local                  9090      -          outbound      EDS              
prometheus_stats                                           -         -          -             STATIC           
sds-grpc                                                   -         -          -             STATIC           
tracing.istio-system.svc.cluster.local                     80        -          outbound      EDS              
tracing.istio-system.svc.cluster.local                     16685     -          outbound      EDS              
xds-grpc                                                   -         -          -             STATIC           
zipkin                                                     -         -          -             STRICT_DNS       
zipkin.istio-system.svc.cluster.local                      9411      -          outbound      EDS              

ADDRESS        PORT  MATCH                                                                    DESTINATION
10.96.0.10     53    ALL                                                                      Cluster: outbound|53||kube-dns.kube-system.svc.cluster.local
0.0.0.0        80    Trans: raw_buffer; App: http/1.1,h2c                                     Route: 80
0.0.0.0        80    ALL                                                                      PassthroughCluster
10.102.38.206  443   ALL                                                                      Cluster: outbound|443||myistio-ingressgateway.istio-ingress.svc.cluster.local
10.109.184.232 443   ALL                                                                      Cluster: outbound|443||istiod.istio-system.svc.cluster.local
10.96.0.1      443   ALL                                                                      Cluster: outbound|443||kubernetes.default.svc.cluster.local
10.96.248.46   443   ALL                                                                      Cluster: outbound|443||istio-ingressgateway.istio-system.svc.cluster.local
10.98.124.246  3000  Trans: raw_buffer; App: http/1.1,h2c                                     Route: grafana.istio-system.svc.cluster.local:3000
10.98.124.246  3000  ALL                                                                      Cluster: outbound|3000||grafana.istio-system.svc.cluster.local
0.0.0.0        8080  Trans: raw_buffer; App: http/1.1,h2c                                     Route: 8080
0.0.0.0        8080  ALL                                                                      PassthroughCluster
0.0.0.0        9090  Trans: raw_buffer; App: http/1.1,h2c                                     Route: 9090
0.0.0.0        9090  ALL                                                                      PassthroughCluster
10.96.0.10     9153  Trans: raw_buffer; App: http/1.1,h2c                                     Route: kube-dns.kube-system.svc.cluster.local:9153
10.96.0.10     9153  ALL                                                                      Cluster: outbound|9153||kube-dns.kube-system.svc.cluster.local
0.0.0.0        9411  Trans: raw_buffer; App: http/1.1,h2c                                     Route: 9411
0.0.0.0        9411  ALL                                                                      PassthroughCluster
10.100.204.154 14250 Trans: raw_buffer; App: http/1.1,h2c                                     Route: jaeger-collector.istio-system.svc.cluster.local:14250
10.100.204.154 14250 ALL                                                                      Cluster: outbound|14250||jaeger-collector.istio-system.svc.cluster.local
10.100.204.154 14268 Trans: raw_buffer; App: http/1.1,h2c                                     Route: jaeger-collector.istio-system.svc.cluster.local:14268
10.100.204.154 14268 ALL                                                                      Cluster: outbound|14268||jaeger-collector.istio-system.svc.cluster.local
0.0.0.0        15001 ALL                                                                      PassthroughCluster
0.0.0.0        15001 Addr: *:15001                                                            Non-HTTP/Non-TCP
0.0.0.0        15006 Addr: *:15006                                                            Non-HTTP/Non-TCP
0.0.0.0        15006 Trans: tls; App: istio-http/1.0,istio-http/1.1,istio-h2; Addr: 0.0.0.0/0 InboundPassthroughClusterIpv4
0.0.0.0        15006 Trans: tls; Addr: 0.0.0.0/0                                              InboundPassthroughClusterIpv4
0.0.0.0        15006 Trans: tls; Addr: *:80                                                   Cluster: inbound|80||
0.0.0.0        15010 Trans: raw_buffer; App: http/1.1,h2c                                     Route: 15010
0.0.0.0        15010 ALL                                                                      PassthroughCluster
10.109.184.232 15012 ALL                                                                      Cluster: outbound|15012||istiod.istio-system.svc.cluster.local
0.0.0.0        15014 Trans: raw_buffer; App: http/1.1,h2c                                     Route: 15014
0.0.0.0        15014 ALL                                                                      PassthroughCluster
0.0.0.0        15021 ALL                                                                      Inline Route: /healthz/ready*
10.102.38.206  15021 Trans: raw_buffer; App: http/1.1,h2c                                     Route: myistio-ingressgateway.istio-ingress.svc.cluster.local:15021
10.102.38.206  15021 ALL                                                                      Cluster: outbound|15021||myistio-ingressgateway.istio-ingress.svc.cluster.local
10.96.248.46   15021 Trans: raw_buffer; App: http/1.1,h2c                                     Route: istio-ingressgateway.istio-system.svc.cluster.local:15021
10.96.248.46   15021 ALL                                                                      Cluster: outbound|15021||istio-ingressgateway.istio-system.svc.cluster.local
0.0.0.0        15090 ALL                                                                      Inline Route: /stats/prometheus*
0.0.0.0        16685 Trans: raw_buffer; App: http/1.1,h2c                                     Route: 16685
0.0.0.0        16685 ALL                                                                      PassthroughCluster
0.0.0.0        20001 Trans: raw_buffer; App: http/1.1,h2c                                     Route: 20001
0.0.0.0        20001 ALL                                                                      PassthroughCluster

NAME                                                             DOMAINS                                                 MATCH                  VIRTUAL SERVICE
myistio-ingressgateway.istio-ingress.svc.cluster.local:15021     *                                                       /*                     
8080                                                             helloworld, helloworld.default + 1 more...              /*                     
kube-dns.kube-system.svc.cluster.local:9153                      *                                                       /*                     
80                                                               istio-ingressgateway.istio-system, 10.96.248.46         /*                     
80                                                               myistio-ingressgateway.istio-ingress, 10.102.38.206     /*                     
80                                                               tracing.istio-system, 10.103.51.183                     /*                     
jaeger-collector.istio-system.svc.cluster.local:14250            *                                                       /*                     
grafana.istio-system.svc.cluster.local:3000                      *                                                       /*                     
istio-ingressgateway.istio-system.svc.cluster.local:15021        *                                                       /*                     
                                                                 *                                                       /stats/prometheus*     
InboundPassthroughClusterIpv4                                    *                                                       /*                     
                                                                 *                                                       /healthz/ready*        
inbound|80||                                                     *                                                       /*                     
jaeger-collector.istio-system.svc.cluster.local:14268            *                                                       /*                     
9090                                                             byeworld.foo, 10.103.187.190                            /*                     
9090                                                             kiali.istio-system, 10.104.141.120                      /*                     
9090                                                             prometheus.istio-system, 10.107.129.0                   /*                     
9411                                                             jaeger-collector.istio-system, 10.100.204.154           /*                     
9411                                                             zipkin.istio-system, 10.104.238.43                      /*                     
15010                                                            istiod.istio-system, 10.109.184.232                     /*                     
15014                                                            istiod.istio-system, 10.109.184.232                     /*                     
16685                                                            tracing.istio-system, 10.103.51.183                     /*                     
20001                                                            kiali.istio-system, 10.104.141.120                      /*                     

RESOURCE NAME     TYPE           STATUS     VALID CERT     SERIAL NUMBER                               NOT AFTER                NOT BEFORE
default           Cert Chain     ACTIVE     true           224526398421470636195992462181330755939     2023-04-23T23:57:50Z     2023-04-22T23:55:50Z
ROOTCA            CA             ACTIVE     true           3144612513681150263454419199256531619       2033-04-17T19:15:16Z     2023-04-20T19:15:16Z
```


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
  conn_handler: warning
  decompression: warning
  dns: warning
  dubbo: warning
  envoy_bug: warning
  ext_authz: warning
  ext_proc: warning
  rocketmq: warning
  file: warning
  filter: warning
  forward_proxy: warning
  grpc: warning
  happy_eyeballs: warning
  hc: warning
  health_checker: warning
  http: warning
  http2: warning
  hystrix: warning
  init: warning
  io: warning
  jwt: warning
  kafka: warning
  key_value_store: warning
  lua: warning
  main: warning
  matcher: warning
  misc: error
  mongo: warning
  multi_connection: warning
  oauth2: warning
  quic: warning
  quic_stream: warning
  pool: warning
  rate_limit_quota: warning
  rbac: warning
  rds: warning
  redis: warning
  router: warning
  runtime: warning
  stats: warning
  secret: warning
  tap: warning
  testing: warning
  thrift: warning
  tracing: warning
  upstream: warning
  udp: warning
  wasm: warning
  websocket: warning
```