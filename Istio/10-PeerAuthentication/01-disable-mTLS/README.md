---
gitea: none
include_toc: true
---

# Description

On this example we disable the mTLS for the service deployed, and observe which is the behavior, and one possible environment where it might be required to disable mTLS.

This example uses the `selector` field to target labels set to the deployments.

Also explores the behavior of accessing an `HTTPS` backend using the tls `STRICT` mode, when using `mTLS` and when `mTLS` is disabled.

To explore the different behaviors, [2 deployments](#deployments) where used, both under the same [Service](#service), and the traffic will be distributed through subsets in the [Destination Rule](#destination-rule) set.

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
peerauthentication.security.istio.io/disable-mtls created
peerauthentication.security.istio.io/force-mtls created
deployment.apps/helloworld-mtls created
deployment.apps/helloworld-nomtls created
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



## Analyze the different behaviours

> **DISCLAIMER**:\
> For some reason, during the packet captures, I required to execute the curl 2 times in order for the output to be updated.\
> During the tests, feel free to perform the curl twice in a row.

This steps will be structured on 3 parts:

- Starting the packet capture.
- Using `curl` to send a request to the destination. This step can also be performed through a web browser.
- Observing the information captured in the packet capture.

All this steps will be performed for each one of the environments, each environment being formed by 2 backend destinations.

Environments:

- mTLS disabled
- mTLS enabled

Backend destinations in each one of the environments:
- HTTP
- HTTPS


### mTLS disabled


#### HTTP

##### Start the packet capture for the port 80

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

### mTLS enabled

#### HTTP

##### Start the packet capture for the port 80

```shell
PORT=80 && MTLS="true" && kubectl exec -n default  "$(kubectl get pod -n default -l app=helloworld -l mtls=${MTLS} -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port ${PORT} -A
```
```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

##### Curl

We can access the service.

```shell
curl 192.168.1.50/http-mTLS
```
```text
<h2>Howdy</h2>
```
##### Reviewing pcap output

Due to mTLS being enabled, the traffic captured is encrypted, and for such we cannot explore the contents of such.

We can notice the following lines `outbound_.8080_.mtls_.helloworld.default.svc.cluster.local`, and further deep in the sea of text `1.1.istio.http/1.1` referring that `mTLS` termination was performed through the HTTP version `HTTP1.1`.

```text
04:21:48.543118 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.40224 > helloworld-mtls-7998d9646b-sv7hp.http: Flags [S], seq 4217286528, win 64800, options [mss 1440,sackOK,TS val 1478647369 ecr 0,nop,wscale 7], length 0
E..<..@.>..x..yX...,. .P.^......... ...........
X"^I........
04:21:48.544529 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.40224 > helloworld-mtls-7998d9646b-sv7hp.http: Flags [.], ack 3797925086, win 507, options [nop,nop,TS val 1478647370 ecr 861329182], length 0
E..4..@.>.....yX...,. .P.^..._.............
X"^J3V..
04:21:48.545045 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.40224 > helloworld-mtls-7998d9646b-sv7hp.http: Flags [P.], seq 0:2216, ack 1, win 507, options [nop,nop,TS val 1478647371 ecr 861329182], length 2216: HTTP
E.....@.>.....yX...,. .P.^..._.......v.....
X"^K3V................~5pO...T`.|..{. .........Q..e .}..,....q...n....=...'.a7....=r.........+.../...,.0...D...?.=..:outbound_.8080_.mtls_.helloworld.default.svc.cluster.local..........
...............#..... ...istio-http/1.1.istio.http/1.1.........................3.&.$... #.g....m............l....`.KE.>J.-.....+........).k.F.@O..)z.l.....8b......}.2....77.?.......J..T0....]..\....R.W.]....x..W....;.[....x...."Wy.Q.{.c.Fo..W%7....
. .].m..>......2\V._(&........;.....&K.b..};R.._A.$)s....2.gC.....d..>Q.x{.uw...s....<|O.:T...d.........j..O...d2..;...S.&. s.l..v..G........B..|g..!....@6fdG..]....=e..>.2..*}%*..>..u..y.B....vq99:....IT..)I5........`......BG...[5m.../7..
v........R.1...l.S2W{M.7.._w..D....j.,.....O-;6.q....<..P....s
..0..:.....Oq..cX..=.k`Q.X.x.E.E`T.<...Y..tPG.:..z.#p.)$..)@...W..g]Q..W......I..:....~..... .....;Y.YG....+.o.,.....8t...l.q.&.........1..w.{.[.U..B...]a	up..8:\....:5......../o.5..[.,+xA(.........
...`.M...>...mor.o........`x\.1......:..s.h..r....Mm*..w.Q. ..d..W..&..0[bi.u.F}4...SP=....j\.H._1....6..f....=.\.$.. pD1.6@.>..4YT.D..e".}=.c..,O..M.eC*?...w..R..LZ..f.._.q..bR.t.-I..=,....%"...*...].m..d5..W...3.k...k.s...[ANc._.....V	...z._.b{I...(r.)..v......H.?......*|./h	A|.l.(2..&-..}	...V....D..........g.vA.P/@...._`...M...}..}kF..g.,.rs7...^.0.:W"....8.(.Rr.O{..#I.d.CL....(.D.....L..4..)I3.F.l..kD..`.x<8Z.`..a'.u.
_.^RMn.w..	..?y...R.T.P.c...9...Q.....w.._.T..;...... .l...?..w!.T.._.,...p
..I.zG....x.^p.........X......7v.'.pp..u....ab^Q_
pS.........B...6....s;......
. ..Q....nRw.\HG.H]......l.....G._..4% .{.<...a..p.5\...0......86..Al........&..;....\.V....d.U.......-.Y.<WH...^..."4..... .._	.ov....V....;.h..d.C.0.&.3...s...-....9.....>....B..v.9...k...]S.)....V]C8.<....0..V..fP.oe..{........	.....8:.{tU..`]...@_.H.t.a...9.}......eE...F..6........!S=....)..W4.;...?..C.... ....t.D..IU....RU.X'V.....t...M.j.'-...^p..1...S.9. ...o.J..8v..C2..%..d.T..GU..-.?...F"`....z.../........s.N....$I..F'0....#........B.4..S.M...#..)...Sx.....E....f.).....m.k.G1..I..$=.Q..^n.[..tn..y4.g.}.7...&..}..JXRk..<...S%r..."]#.......O..;.Rt......2.v..~e.E.{t.F.b...4..-:......6..CrE#.....^]~.k5.@..*.^.K..G.k..(dc.#L..z...L..8..._........d..gXl.......! |r?....%Z&!]n....C7...c.([6u....
04:21:48.550098 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.40224 > helloworld-mtls-7998d9646b-sv7hp.http: Flags [.], ack 219, win 506, options [nop,nop,TS val 1478647376 ecr 861329188], length 0
E..4..@.>..|..yX...,. .P.^.)._.............
X"^P3V.$
04:21:48.551427 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.40224 > helloworld-mtls-7998d9646b-sv7hp.http: Flags [P.], seq 2216:2280, ack 219, win 506, options [nop,nop,TS val 1478647378 ecr 861329188], length 64: HTTP
E..t..@.>..;..yX...,. .P.^.)._.......7.....
X"^R3V.$..........5k)...o...^D......3..........WC.|...@...zwS...z.@yA.c
04:21:48.551870 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.40224 > helloworld-mtls-7998d9646b-sv7hp.http: Flags [P.], seq 2280:3959, ack 219, win 506, options [nop,nop,TS val 1478647378 ecr 861329188], length 1679: HTTP
E.....@.>.....yX...,. .P.^.i._.......].....
X"^R3V.$......zb5...o.....x.....a..-....B^4...K.m.
..Z..z..(.f3aG......r...$9
```



#### HTTPS

##### Start the packet capture for the port 443

```shell
PORT=443 && MTLS="true" && kubectl exec -n default  "$(kubectl get pod -n default -l app=helloworld -l mtls=${MTLS} -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port ${PORT} -A
```
```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

##### Curl

On this scenario, we met a fatal error, not allowing us to access the service, unlike the previous attempts.

From my understanding, not only from this interaction, but from investigating through Istio forums (yet I don't have the link handy, so take this words with some grains of salt), **the traffic cannot be double terminated**, for such if we have an `HTTPS` backend, we might require to disable `mTLS` in order to communicate with it. We also would need to set a [Destination Rule like we did further above](#destination-rule), to specify that the traffic must be terminated with the backend (`tls.mode: STRICT`).

Yet this depends on which would be our architecture, due also being able to set up [TLS Passthrough](../../02-Traffic_management/11-TLS-PASSTHROUGH), or use a [TCP Forwarding](../../02-Traffic_management/10-TCP-FORWARDING).

```shell
curl 192.168.1.50/https-mTLS
```
```text
upstream connect error or disconnect/reset before headers. reset reason: connection termination
```

##### Reviewing pcap output


Not much to highlight as there isn't much available text for us to be able to read.

```text
04:22:15.813163 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.50576 > helloworld-mtls-7998d9646b-sv7hp.https: Flags [S], seq 693161527, win 64800, options [mss 1440,sackOK,TS val 1478674639 ecr 0,nop,wscale 7], length 0
E..<..@.>.}Z..yX...,....)P.7....... ...........
X"..........
04:22:15.814619 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.50576 > helloworld-mtls-7998d9646b-sv7hp.https: Flags [.], ack 609580424, win 507, options [nop,nop,TS val 1478674641 ecr 861356452], length 0
E..4..@.>.}a..yX...,....)P.8$Uu......x.....
X"..3WA.
04:22:15.815126 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.50576 > helloworld-mtls-7998d9646b-sv7hp.https: Flags [P.], seq 0:246, ack 1, win 507, options [nop,nop,TS val 1478674641 ecr 861356452], length 246
E..*..@.>.|j..yX...,....)P.8$Uu............
X"..3WA.............#j..S..(.j....4\v.h_ N......S.O e....U.....oM.j.....l...t......T.........+.../...,.0..............
...............#..... ...istio-http/1.1.istio.http/1.1.........................3.&.$... ..t.i.=..1...[i..
FQF.....8d..}..-.....+.......
04:22:15.831747 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.50576 > helloworld-mtls-7998d9646b-sv7hp.https: Flags [.], ack 2165, win 499, options [nop,nop,TS val 1478674658 ecr 861356470], length 0
E..4..@.>.}_..yX...,....)P..$U}............
X"..3WA.
04:22:15.834886 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.50576 > helloworld-mtls-7998d9646b-sv7hp.https: Flags [P.], seq 246:318, ack 2165, win 501, options [nop,nop,TS val 1478674661 ecr 861356470], length 72
E..|..@.>.}...yX...,....)P..$U}.....+......
X"..3WA...........=d..Vv.s..."..Dc.p...T...s...3........i.'-Sc..0p|)...!. T~...)
04:22:15.835307 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.50576 > helloworld-mtls-7998d9646b-sv7hp.https: Flags [P.], seq 318:1999, ack 2165, win 501, options [nop,nop,TS val 1478674661 ecr 861356470], length 1681
E.....@.>.v...yX...,....)P.v$U}......_.....
X"..3WA.......7..t...*.U.....,...]...l..=.x....jH..*......[..._...._..l..+......T..9.}$CO.[...b.Fx0...X.2.......V.U.%.^.%.}?...	...$..G.\0G..=.9.X....jA...ks^r.H.*H.....2H........Im
...........@..D!O0...a..G.i/1..W-.....A..yd..`...h.'Y...&.Po..T.4..B........$..t...M.....D..Y..6z.....8I-....e.3.....4.$Y.C_R...'V......C...&.\....."...U.[T....nW..}......!......L..j..ov...~.....r..
B..B.gRp
R..xLTm..af_.X.2.......|.,.Wi.....F@.0.'...
.>.8.'t.....r.Xi....#*..l.bO.V.......G.[:....7.2.(U....R,#.>!..<.o..w..R|.T..:_..i.. iJs.-.>...B..~.mOH0+N.....-.b...5.._.9%....u&..y.S...8A...*.=....MJS.m........u..Ic....s}Y....{.8d.....<..P-;V[......\.....+..S.8k..r&...dT..K..y].t..3..BU,.<......:IH......-..\j.g...\:..[........(.S......"..0|-.p"Z..:..>6..b..x.....M..;K2AT|Ah.....3z.+..><.&........)E.C ..4....X1.p} .@...@n.........\..R...H........5...+h-...q.|.(....]o..	jw..(....=.. 
..+(nY{......6..@..c.^.........o..:.V|..0....	N*..e*...G.,{...wb...-y..g.k7...,tI.|..........H....4E.2..!b........K..&q1..0.us|z...he/.T+6b.}.L........q...F....nTs.Vp!.........W.F..j...X
./.gIv..6G(Ze.h`.......<..w...........@!E..N.>..^.[..IO$T.]6.D..K%m.....LD @.
	.......f!O....5	...K...Y..}.I.o.]q0`..H&...d.aZ.1...P.......R.. C.jfM....;9........y.h.E)...r.....B....#.\......Q..fX..~....ixh	..t.q.y....BkR.nr.k5.`@.8..Z5_Gl.l.
...'t....q......	....t8......_`.....:4>.H....S.e/=!.V..	.6...X.o..K.H...S@.3...a.....].j-.$Q.6..{..kr.....=a.....-.......-2....D.....&......:..y.DJQ.0....E....,Uc......H..6.`.u.....).f..R.xp.H....(.c.9..a.*.P$d..KD..;.x.$,....L.......`..x...p.[..d...z.,jV.[0....j.r."\..._....[......].o..5.Q*.Y.....b0.......-..B...^..)9....S.l...Ek?..~9......`....^...../G{Q14......7......SVV.A.>8..].
04:22:15.835726 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.50576 > helloworld-mtls-7998d9646b-sv7hp.https: Flags [.], ack 2189, win 501, options [nop,nop,TS val 1478674662 ecr 861356474], length 0
E..4..@.>.}[..yX...,....)P..$U~............
X"..3WA.
04:22:15.835912 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.505
```


## Cleanup

```shell
kubectl delete -f ./
```

```text
service "helloworld" deleted
peerauthentication.security.istio.io "disable-mtls" deleted
peerauthentication.security.istio.io "force-mtls" deleted
deployment.apps "helloworld-mtls" deleted
deployment.apps "helloworld-nomtls" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
destinationrule.networking.istio.io "helloworld.default.svc.cluster.local" deleted
```


# Links of Interest

- https://istio.io/latest/docs/reference/config/security/peer_authentication/#PeerAuthentication-MutualTLS-Mode

- https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/

- https://istio.io/latest/docs/concepts/security/#mutual-tls-authentication

- https://istio.io/latest/docs/reference/config/security/peer_authentication/
