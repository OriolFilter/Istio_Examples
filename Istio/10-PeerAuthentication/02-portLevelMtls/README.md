---
gitea: none
include_toc: true
---

# Based on

- [01-disable-mTLS](../01-disable-mTLS)

# Description

Based on the previous example that disabled mTLS, and explored how it affected the behavior of the services, on `HTTP` and `HTTPS` backends, this example aims to, through the usage of `portLevelMtls`, configure the `mTLS` behavior based on the destination port.

Through this, we can apply multiple `mTLS` behaviors under a single deployment, unlike the [previous example](../01-disable-mTLS) that required to create 2 different deployments under a single service, and as well implement `Destination Rules` as well of `subsets` to route the traffic between the 2 deployments.  

> **Note:**\
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-nginx-demo)

# Configuration

## Gateway

Listens for `HTTP` traffic without limiting any host.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: helloworld-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

## Virtual Service

Without limiting to any host, listens for traffic at port 80, and only has a very specific URL paths available to match.

- /http-mTLS
- /https-mTLS
- /http-no-mTLS
- /https-no-mTLS

Depending on the path used, the traffic will be distributed between 2 subsets from the same service:

- mtls
- nomtls

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-vs
spec:
  hosts:
    - "*"
  gateways:
    - helloworld-gateway
  http:
    - name: http-mTLS
      match:
        - port: 80
          uri:
            exact: "/http-mTLS"
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8080
            subset: mtls
      rewrite:
        uri: "/"
    - name: https-mTLS
      match:
        - port: 80
          uri:
           exact: "/https-mTLS"
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8443
            subset: mtls
      rewrite:
        uri: "/"
    - name: http-no-mTLS
      match:
        - port: 80
          uri:
            exact: "/http-no-mTLS"
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8080
            subset: nomtls
      rewrite:
        uri: "/"
    - name: https-no-mTLS
      match:
        - port: 80
          uri:
            exact: "/https-no-mTLS"
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8443
            subset: nomtls
      rewrite:
        uri: "/"
```

## Destination Rule

Interfering with the service URL `helloworld.default.svc.cluster.local`, it specifies 2 subsets:

- mtls
- nomtls

Additionally, specifies that the traffic with port destination 8443, will attempt to proceed with TLS termination, as it is required to connect with an `HTTPS` backend. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld.default.svc.cluster.local
spec:
  host: helloworld.default.svc.cluster.local
  subsets:
    - name: mtls
      labels:
        mtls: "true"

    - name: nomtls
      labels:
        mtls: "false"

  trafficPolicy:
    portLevelSettings:
      - port:
          number: 8443
        tls:
          mode: SIMPLE # Required for https backend
```

## Service

The service will forward incoming traffic from the service port `8443`, that will be forwarded towards the port `443` from the deployment, which contains an `HTTPS` service.

Also listens for `HTTP` traffic at the port `8080`, and will be forwarded to the deployment port `80`.

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
      
    - port: 8443
      name: https
      targetPort: 443
      protocol: TCP
      appProtocol: https
  selector:
    app: helloworld
```

## Deployments

There's been configured 2 deployments with the same service and settings, besides the label `mtls`, which will contain `true` or `false` based on the deployment.

This label is used for the [Destination Rule](#destination-rule) to distribute the traffic between the 2 deployments under the same service.

> **Note:**\
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-nginx-demo)

### helloworld-mtls

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-mtls
  labels:
    app: helloworld
    mtls: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      mtls: "true"
  template:
    metadata:
      labels:
        app: helloworld
        mtls: "true"
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

### helloworld-nomtls

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-nomtls
  labels:
    app: helloworld
    mtls: "false"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      mtls: "false"
  template:
    metadata:
      labels:
        app: helloworld
        mtls: "false"
    spec:
      containers:
        - name: helloworld-nomtls
          image: oriolfilter/https-nginx-demo
          resources:
            requests:
              cpu: "100m"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
            - containerPort: 443
```

## PeerAuthentications

Deployed 2 Peer Authentication rules, which use the `selector` field to target the deployments.

Both point to the same application, yet also specify the `mtls` label set in the deployments above, allowing the rules to target each deployment individually.

These rules are deployed in the `default` namespace.

### disable-mtls

This rule will disable `mTLS` for that deployment.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: disable-mtls
  namespace: default
spec:
  selector:
    matchLabels:
      app: helloworld
      mtls: "false"
  mtls:
    mode: DISABLE
```

### force-mtls

This rule forces the deployment to communicate exclusively through `mTLS`, in case this rule is not endorsed, the traffic won't be allowed to proceed further.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: force-mtls
  namespace: default
spec:
  selector:
    matchLabels:
      app: helloworld
      mtls: "true"
  mtls:
    mode: STRICT
```

# Walkthrough

## Deploy resources

```shell
kubectl apply -f ./
```
```text
service/helloworld created
peerauthentication.security.istio.io/helloworld-mtls created
deployment.apps/helloworld created
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
destinationrule.networking.istio.io/helloworld.default.svc.cluster.local created
```

## Get LB IP

```shell
kubectl get svc -l istio=ingressgateway -A
```
```text
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```

## Test resources and analyze behaviors

> **DISCLAIMER**:\
> For some reason, during the packet captures, I required to execute the curl 2 times in order for the output to be updated.\
> During the tests, feel free to perform the curl twice in a row.

### HTTP

#### Start the packet capture for the port 80

Start the packet capture and proceed with another shell or browser to send traffic requests to the right destination.

```shell
PORT=80 && MTLS="false" && kubectl exec -n default  "$(kubectl get pod -n default -l app=helloworld -l mtls=${MTLS} -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port ${PORT} -A
```
```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

##### Curl

Nothing to higlight so far, we can access the service.

```shell
curl 192.168.1.50/http-no-mTLS
```
```text
<h2>Howdy</h2>
```

##### Reviewing pcap output

Due to having the mTLS disabled, the traffic is not encrypted, and for such we can see its context in plain text.

This scenario should be avoided unless it is required due the application being used, as mTLS allows an extra layer of security. 

```text
04:25:47.757900 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.60966 > helloworld-nomtls-66d8499c5c-298vw.http: Flags [P.], seq 3134140617:3134142280, ack 2649160847, win 501, options [nop,nop,TS val 1425864700 ecr 2534833629], length 1663: HTTP: GET / HTTP/1.1
E....t@.?.....yX..yx.&.P..0.........Q......
T.....}.GET / HTTP/1.1
host: 192.168.1.50
user-agent: curl/8.0.1
accept: */*
x-forwarded-for: 192.168.1.10
x-forwarded-proto: http
x-envoy-internal: true
x-request-id: 65b60be7-da98-48f3-9ed6-13112cdd14f0
x-envoy-decorator-operation: helloworld.default.svc.cluster.local:8080/http-no-mTLS
x-envoy-peer-metadata: ChQKDkFQUF9DT05UQUlORVJTEgIaAAoaCgpDTFVTVEVSX0lEEgwaCkt1YmVybmV0ZXMKHwoMSU5TVEFOQ0VfSVBTEg8aDTE3Mi4xNy4xMjEuODgKGQoNSVNUSU9fVkVSU0lPThIIGgYxLjE3LjIKnAMKBkxBQkVMUxKRAyqOAwodCgNhcHASFhoUaXN0aW8taW5ncmVzc2dhdGV3YXkKEwoFY2hhcnQSChoIZ2F0ZXdheXMKFAoIaGVyaXRhZ2USCBoGVGlsbGVyCjYKKWluc3RhbGwub3BlcmF0b3IuaXN0aW8uaW8vb3duaW5nLXJlc291cmNlEgkaB3Vua25vd24KGQoFaXN0aW8SEBoOaW5ncmVzc2dhdGV3YXkKGQoMaXN0aW8uaW8vcmV2EgkaB2RlZmF1bHQKMAobb3BlcmF0b3IuaXN0aW8uaW8vY29tcG9uZW50EhEaD0luZ3Jlc3NHYXRld2F5cwoSCgdyZWxlYXNlEgcaBWlzdGlvCjkKH3NlcnZpY2UuaXN0aW8uaW8vY2Fub25pY2FsLW5hbWUSFhoUaXN0aW8taW5ncmVzc2dhdGV3YXkKLwojc2VydmljZS5pc3Rpby5pby9jYW5vbmljYWwtcmV2aXNpb24SCBoGbGF0ZXN0CiIKF3NpZGVjYXIuaXN0aW8uaW8vaW5qZWN0EgcaBWZhbHNlChoKB01FU0hfSUQSDxoNY2x1c3Rlci5sb2NhbAovCgROQU1FEicaJWlzdGlvLWluZ3Jlc3NnYXRld2F5LTg2NGRiOTZjNDctZjZscWQKGwoJTkFNRVNQQUNFEg4aDGlzdGlvLXN5c3RlbQpdCgVPV05FUhJUGlJrdWJlcm5ldGVzOi8vYXBpcy9hcHBzL3YxL25hbWVzcGFjZXMvaXN0aW8tc3lzdGVtL2RlcGxveW1lbnRzL2lzdGlvLWluZ3Jlc3NnYXRld2F5ChcKEVBMQVRGT1JNX01FVEFEQVRBEgIqAAonCg1XT1JLTE9BRF9OQU1FEhYaFGlzdGlvLWluZ3Jlc3NnYXRld2F5
x-envoy-peer-metadata-id: router~172.17.121.88~istio-ingressgateway-864db96c47-f6lqd.istio-system~istio-system.svc.cluster.local
x-envoy-attempt-count: 1
x-envoy-original-path: /http-no-mTLS
x-b3-traceid: 36e7d48757f2ce26eaa6e1959f3b1221
x-b3-spanid: eaa6e1959f3b1221
x-b3-sampled: 0
```

#### HTTPS

##### Start the packet capture for the port 443

```shell
PORT=443 && MTLS="false" && kubectl exec -n default  "$(kubectl get pod -n default -l app=helloworld -l mtls=${MTLS} -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port ${PORT} -A
```
```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```


##### Curl

So good so far.

```shell
curl 192.168.1.50/https-no-mTLS
```
```text
<h2>Howdy</h2>
```

##### Reviewing pcap output

Due to the configuration set in the [Destination Rule](#destination-rule), where we set the `tls.mode` setting to `SIMPLE`, the traffic will be TLS terminated with the backend.

For such, the traffic captured is encrypted, even tho we displayed the `mTLS` configuration for this deployment.  

Yet, there are still a couple readable lines, where we can see that the request was initialized by the host `stio-ingressgateway.istio-system.svc.cluster.local`, through the egress port `39884`, using as destination `helloworld-nomtls-66d8499c5c-298vw`, and the port defined with name `https`, which, if we reviewed the configuration from the [Service](#service), we would observe that it was the port `8443`.

```text
k 496, win 505, options [nop,nop,TS val 1425943341 ecr 2534945802], length 0
E..4..@.?.....yX..yx.......;........K......
T.+-..4

04:27:06.400101 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.39884 > helloworld-nomtls-66d8499c5c-298vw.https: Flags [.], ack 809, win 503, options [nop,nop,TS val 1425943342 ecr 2534945803], length 0
E..4..@.?.....yX..yx.......;........K......
T.+...4.
04:27:13.439290 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.39826 > helloworld-nomtls-66d8499c5c-298vw.https: Flags [P.], seq 1997:3684, ack 2200, win 501, options [nop,nop,TS val 1425950381 ecr 2534942622], length 1687
E.....@.?..+..yX..yx....pI.+,U+.....Q......
T.F...'......,.SuD..a....`..]....j..v[tF$y.<......&..m.E.p.Y.-....w..V..T.....g..a~$.Q'.c#......qj..8.|......M.J.....\".>...R...M..k|..f^,.....E.....+.Y.6..].u.g<r...0.eE...QSM	0Q...05......y......h6fbW.HdFp....../..(F\.U.pSn...2 .-.X/.8...P....~4anH.h....e....../.3@(....x...{.4.j@[.....P.6.......%.M.EGo.Q~@.
Z............/$..@....&.8..... f...ip.z]....p..}.....f.=......'......Koz.3..d.@..;....)}...>.m....Z..~o.IL.......D.]h.G.... .....F/..V......}.v.^N.P.C.G.......1..T.....w....?..]:........D...;q?...W..cI.).O......3..X14P..B.).',.N...B.../q..)\.. GW."....	.`.....[9.IS......1y.J]...d..}...B.n...C.........e6..B..[w.\.3.l.HU....5%......p.irW.@s..!1\u./.~..[.g..W.........'W..,m};._../S2\..c.9..8..rg"f..35a.A.;..T....>`..Zv.L.8....hZ".*r...0..*.%K.?..	.P]DKve/E.J.....\....t.e.9#-..3.$).....Q.Z.....m].".	q. *.OW...f.=l...K.o:.D.......+.a..h?{h.?..T.....7\N.....M.`..Ob1`.....3d.aq..0...q.r.*j....KE./.O...T%..r.......'..9.W1J^^TU8.$...Y."~..~ZH.......G..?......Q4..=|.{.d/..^_....`.pjJ+p.........R."..Y-.`1....{....k...]ib.+m.....6..k...U.P.T........wU...}......`.z..#..[1.@9.z+R.3pAW).......m...Px4..9^	X..ux.EVO.o.%./+.....|4..!s......g.1...9%.... B.....{.6..].-?.../..n..y...2..sLc..|x.
,.t..'...7.............|...........?..&}........@...=.|#.+...........u.3....m.X.....	QrW?............u`-k....Q.o^{........$..h.....R.#...k...o.7~.*.tE.C...I<"......k..czN.DJ.y...R.....hx.he.r}0.82....6.J...)..3.f.G=Ky|f.L.).=.hlN!..D..J..g.V.?.......#...fQ..d.......9.9.-....j..O...Pd..E.da/..b} .}.Qx.......I..[+....>.5....p.9....K2M s(.a..K6.]..m.?...%..</.S.9......[.P./.1.I. ...k.'.`V........^O.....q..	<...H..=mZZ...........@.VR..x.....U..t....s!.......M.m.........u...:.....V.1X...2.T..~...
04:27:13.440468 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.39826 > helloworld-nomtls-66d8499c5c-298vw.https: Flags [.], ack 2513, win 501, options [nop,nop,TS val 1425950382 ecr 2534952843], length 0
E..4..@.?.....yX..yx....pI..,U,.....K......
T.F...O.
04:27:20.932653 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.40126 > helloworld-nomtls-66d8499c5c-298vw.https: Flags [S], seq 3645561416, win 64800, options [mss 1440,sackOK,TS val 1425957874 ecr 0,nop,wscale 7], length 0
E..<..@.?.f>..yX..yx.....J.H....... K".........
T.c.........
04:27:20.933038 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.40126 > helloworld-nomtls-66d8499c5c-298vw.https: Flags [.], ack 840930767, win 507, options [nop,nop,TS val 1425957875 ecr 2534960336], length 0
E..4..@.?.fE..yX..yx.....J.I2.......K......
T.c...l.
04:27:20.933916 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.40126 > helloworld-nomtls-66d8499c5c-298vw.https: Flags [P.], seq 0:517, ack 1, win 507, options [nop,nop,TS val 1425957876 ecr 2534960336], length 517
E..9..@.?.d?..yX..yx.....J.I2.......M......
T.c...l..............#.."H..\..\A*...5.../m.....wV. ;.......>..`..k.t.b.O.U
e(?.X...........+.../...,.0..............
...............#..... ...istio-http/1.1.istio.http/1.1.........................3.&.$... J7.y.............
..<.Ma.v}.*3LI.-.....+........................)......./.....`.............3.. .[....N.,......i.9;.9V9A..1..J.......W.....o.%.%.<uep.Z"X...6...;|.........f.5AyieJ...+..q...T......x....jO.T$.D!x.pe.....D,.P1.. .a..t..r.x#.J.z...y.q...i:....43..3[/;..P0..\*>#ev..f.....! ........FHc..r...6...e.'J.&..T.p
04:27:20.937464 IP 172-17-1
```


## Cleanup

```shell
kubectl delete -f ./
```

```text
service "helloworld" deleted
peerauthentication.security.istio.io "helloworld-mtls" deleted
deployment.apps "helloworld" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
destinationrule.networking.istio.io "helloworld.default.svc.cluster.local" deleted
```


# Links of Interest

- https://istio.io/latest/docs/reference/config/security/peer_authentication/#PeerAuthentication-MutualTLS-Mode

- https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/

- https://istio.io/latest/docs/concepts/security/#mutual-tls-authentication

- https://istio.io/latest/docs/reference/config/security/peer_authentication/
