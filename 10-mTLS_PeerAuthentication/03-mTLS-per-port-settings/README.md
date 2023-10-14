---
gitea: none
include_toc: true
---

# Based on

- [02-disable-mTLS](../02-disable-mTLS)

# Description

Based on the previous example that disabled mTLS, and explored how it affected the behavior of the services, on `HTTP` and `HTTPS` backends, this example aims to, through the usage of `portLevelMtls`, configure the `mTLS` behavior based on the destination port.

Through this, we can apply multiple `mTLS` behaviors under a single deployment, unlike the [previous example](../02-disable-mTLS) that required to create 2 different deployments under a single service, and as well implement `Destination Rules` as well of `subsets` to route the traffic between the 2 deployments.  

> **Note:**\
> For more information about the image used refer to [here](https://hub.docker.com/r/oriolfilter/https-nginx-demo)

# Configuration

## Gateway

Listens for `HTTP` traffic at the port `80` without limiting to any host.

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

The path `/http` will be routed to the `HTTP` service set in our backend.

The path `/http` will be routed to the `HTTPS` service set in our backend.


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
            exact: "/http"
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8080
      rewrite:
        uri: "/"
    - name: https-mTLS
      match:
        - port: 80
          uri:
            exact: "/https"
      route:
        - destination:
            host: helloworld.default.svc.cluster.local
            port:
              number: 8443
      rewrite:
        uri: "/"
```

## Destination Rule

Interfering with the service URL `helloworld.default.svc.cluster.local`, the traffic with port destination `8443`, will attempt to proceed with TLS termination, as it is required to connect with an `HTTPS` backend. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld.default.svc.cluster.local
spec:
  host: helloworld.default.svc.cluster.local
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

## PeerAuthentication


Deployed a rule that sets a "global" mTLS mode to `STRICT`, meaning that the traffic require mTLS termination in order to proceed further with the request.

Also, at a specific port configuration, the port `443` has the mTLS mode disabled, as the deployment contains an `HTTPS` service we required to disable it in order of the request to be successful.

Through the use of the `selector.matchLabels` field, we targeted our deployment pods, limiting the target of this rule.

> **Note**:\
> In order to use the `portLevelMtls` field, the selector field is required, otherwise it won't take effect.\
> For more information regarding this behavior, refer to the [official Istio documentation regarding PeerAuthentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/#PeerAuthentication)

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: helloworld-mtls
  namespace: default
spec:
  selector:
    matchLabels:
      app: helloworld
  mtls:
    mode: STRICT
  portLevelMtls:
    443:
      mode: DISABLE
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



[//]: # (> **DISCLAIMER**:\)

[//]: # (> For some reason, during the packet captures, I required to execute the curl 2 times in order for the output to be updated.\)

[//]: # (> During the tests, feel free to perform the curl twice in a row.)

### HTTP

#### Start the packet capture for the port 80

Start the packet capture and proceed with another shell or browser to send traffic requests to the right destination.

```shell
PORT=80 && kubectl exec -n default  "$(kubectl get pod -n default -l app=helloworld -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port ${PORT} -A
```
```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

##### Curl

Nothing to higlight so far, we can access the service.

```shell
curl 192.168.1.50/http
```
```text
<h2>Howdy</h2>
```

##### Reviewing pcap output

As we can observe, the traffic is encrypted, proving that the mTLS is taking effect terminating the connection with the `HTTP` backend.

```text
02:00:10.511593 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.54396 > helloworld-6798765f88-76r6c.http: Flags [S], seq 3999274711, win 64800, options [mss 1440,sackOK,TS val 2646430461 ecr 0,nop,wscale 7], length 0
E..<..@.>..I..yX...=.|.P.`......... .z.........
..R.........
02:00:10.512773 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.54396 > helloworld-6798765f88-76r6c.http: Flags [.], ack 134781521, win 507, options [nop,nop,TS val 2646430462 ecr 2887117842], length 0
E..4..@.>..P..yX...=.|.P.`.....Q...........
..R.....
02:00:10.512988 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.54396 > helloworld-6798765f88-76r6c.http: Flags [P.], seq 0:517, ack 1, win 507, options [nop,nop,TS val 2646430462 ecr 2887117842], length 517: HTTP
E..9..@.>..J..yX...=.|.P.`.....Q...........
..R.................a7.i..v{
.Nr.0.Yex..C7..k.6...d .......z._ikW3.C.H.....5..Yk.&.c.........+.../...,.0.......;.9..6outbound_.8080_._.helloworld.default.svc.cluster.local..........
...............#..... ...istio-http/1.1.istio.http/1.1.........................3.&.$... .....M4...^9V........d_..+J."..Z.-.....+.......................................................................................................................................................................................................................
02:00:10.530088 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.54396 > helloworld-6798765f88-76r6c.http: Flags [.], ack 2164, win 499, options [nop,nop,TS val 2646430479 ecr 2887117859], length 0
E..4..@.>..N..yX...=.|.P.`...........3.....
..S....#
02:00:10.551166 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.54396 > helloworld-6798765f88-76r6c.http: Flags [P.], seq 517:2501, ack 2164, win 501, options [nop,nop,TS val 2646430500 ecr 2887117859], length 1984: HTTP
E.....@.>.....yX...=.|.P.`.................
..S$...#...........D.W_....+..v{..Q.3....^..m~..aU.+~t..%b....O.X|.).....=.w.z...'....`.2._...7.N..9.y..V.y.&..*vBx..z)B.g.D...1...x....V.J.*!....5.......#.......9.....V.Y..kes.&:+..j;.X5C.I...h+.SO0V....A..b,?.d.@YOy.`x.......o.EcTf}n.0....!N..Qh?.uK#?.Nx..q.&..9|?.)".qpg.]..O2.;O.x...J...$0.......I......1.X2.......2..=.UG.h'pA.CKX........
.	. 7W.v...q..?IW.M:..'d....!2..Y......I.P..).Y..~..>.:k..y..Z?....w.D.Y<s.	3tg.wv...(.8.'.Nd..T...U...\.L.rM9..v........b...d..3...`..2....*. ...Qh...5.J
].	G...3..s..9.4....M..J..s..u.n..j....@.;.8.-LZD#t...z.;..2..M+.2.#.......E.....F.+.u.1....	l4..`......@..{......[.{
%[.R.e.v........@V....1...f*O$..V
pu........Zl...@.%......b..................}.J.....H...h?@[...T>..M.B.MXH.HDa...(.].B......k...{..c&.0...S3..]..2.a\.......?.#..........]3...~...Q|w...l."Z;.4..!.1..,X.>YE..3Yw..9.....#|.....[`...qq..@v.m..1.|V.j$t.C..&.Ww...5e....?.|Q."..obR.a...^...D8;...=.1.....S[.90...ss....-.@..q.JI........$.8..)skW.....G....3:.qb..#/....#...'/n.~F...(Y[.k..EEz}...cgR..6...P..)'.X..e..z....Tv0>....l.t.O=D.vc..}.a.ct.....E/.*..]-`.....O.hY..j..u...."(QZ.^.......f.1.LZ.O.L.9}..m.1_sC....x.*`D..ny.......):.V.."n..t.0....T.S..u[._v%...q`._.....W.w_.q...........O.:J.....[S.a$...l.[.	..cP..zF..~..+..|.....l..	[.l.."/
.....D6f....9:..i............N........o.....;...%v.0@...n^..."OSN.o*.:ap.C#C.Hc..r..MD.
.-..2....
..`...."..I...Wh9.L...r:.4M...b+q
...8f...*.^.K.k.?7:.\..O...	..cD.<t.%$.U..^(..............J.Y..0.[..Z..g.Ok..6/.{.-zjY.K2.Z...)\..RH..*i...=d..z....Dk.n7.0S.).6..D!Y... 2.DnP.9...Gz.:)..D.1.......*...(+...+G)...e.......].qi..eFO..h[.`..4..uk..%..U.........F.......-X.)g..w.=..Q/.f...?+.[\..9.......UW.]..y...e.....ENDG....M.K......~.-.'.....&.-7...k...M..y.8..&.yq.m...`.E...X..67}..L.........Y,F.iS...X.f..Fa.l.'.u.&.x.....2...c.....aw
......}..~.!....pj....m
....5.V......o....{U..M.......q.%.E..$j......Fo.|m......pR...r...v.P<V.!....;....	...r....&....D...,{.U...?,F..(........y}..G......sfFE....9o...q>.c........jM....;......k....
02:00:10.551752 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.54396 > helloworld-6798765f88-76r6c.http: Flags [P.], seq 2501:4170, ack 2164, wi
```

#### HTTPS

##### Start the packet capture for the port 443

```shell
PORT=443 && kubectl exec -n default  "$(kubectl get pod -n default -l app=helloworld -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port ${PORT} -A
```
```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```


##### Curl

Even tho, we have set in the [PeerAuthentication configuration](#peerauthentication) mode to `STRICT`, unlike in the [previous example](../02-disable-mTLS/#https-1), where the mode was also set to `STRICT`, in this example we configured the `portLevelMtls` field for the port `443`, successfully disabling `mTLS` for this port, and allowing to proceed with the request towards the `HTTPS` backend; which was performed without the need of disabling `mTLS` for the whole deployment. 

```shell
curl 192.168.1.50/https
```
```text
<h2>Howdy</h2>
```

##### Reviewing pcap output

Due to the configuration set in the [Destination Rule](#destination-rule), where we set the `tls.mode` setting to `SIMPLE`, the traffic will be TLS terminated with the backend.

For such, the traffic captured is encrypted, even tho we displayed the `mTLS` configuration for this deployment.  

Yet, there are still a couple readable lines, where we can see that the request was initialized by the host `stio-ingressgateway.istio-system.svc.cluster.local`, through the egress port `39884`, using as destination `helloworld-nomtls-66d8499c5c-298vw`, and the port defined with name `https`, which, if we reviewed the configuration from the [Service](#service), we would observe that it was the port `8443`.

```text
02:02:41.616839 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.58752 > helloworld-6798765f88-76r6c.https: Flags [S], seq 1052122243, win 64800, options [mss 1440,sackOK,TS val 2646581565 ecr 0,nop,wscale 7], length 0
E..<.y@.>.....yX...=....>.......... ...........
...=........
02:02:41.618256 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.58752 > helloworld-6798765f88-76r6c.https: Flags [.], ack 1254443190, win 507, options [nop,nop,TS val 2646581567 ecr 2887268947], length 0
E..4.z@.>.....yX...=....>...J.H............
...?..:S
02:02:41.618902 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.58752 > helloworld-6798765f88-76r6c.https: Flags [P.], seq 0:246, ack 1, win 507, options [nop,nop,TS val 2646581568 ecr 2887268947], length 246
E..*.{@.>.....yX...=....>...J.H.....T......
...@..:S............
.B.L(....I....`O.#.$-..f..y.'. :.&.....1oX.i.J.W.CD.-.l.|...y...........+.../...,.0..............
...............#..... ...istio-http/1.1.istio.http/1.1.........................3.&.$... .vw..q|H[6.HQp.zn[. m...M0yL..]g.-.....+.......
02:02:41.637813 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.58752 > helloworld-6798765f88-76r6c.https: Flags [.], ack 1377, win 501, options [nop,nop,TS val 2646581587 ecr 2887268967], length 0
E..4.|@.>.....yX...=....>..zJ.N......4.....
...S..:g
02:02:41.641084 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.58752 > helloworld-6798765f88-76r6c.https: Flags [P.], seq 246:310, ack 1377, win 501, options [nop,nop,TS val 2646581590 ecr 2887268967], length 64
E..t.}@.>..M..yX...=....>..zJ.N............
...V..:g..........5D\..yfI.....]iyu.:........m!Ev.....*..-..`.'*.......g
02:02:41.642627 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.58752 > helloworld-6798765f88-76r6c.https: Flags [.], ack 1632, win 501, options [nop,nop,TS val 2646581592 ecr 2887268972], length 0
E..4.~@.>.....yX...=....>...J.O............
...X..:l
02:02:41.642884 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.58752 > helloworld-6798765f88-76r6c.https: Flags [.], ack 1887, win 501, options [nop,nop,TS val 2646581592 ecr 2887268972], length 0
E..4..@.>.....yX...=....>...J.P............
...X..:l
02:02:41.643146 IP 172-17-121-88.istio-ingressgateway.istio-system.svc.cluster.local.58752 > helloworld-6798765f88-76r6c.https: Flags [P.], seq 310:1981, ack 1887, win 501, options [nop,nop,TS val 2646581592 ecr 2887268972], length 1671
E.....@.>.....yX...=....>...J.P......f.....
...X..:l...............*t\o.....z^=.=\....cq..../.9eKL..`.C....."....q{...*..0;^n7.o:,...a..-.W8:.1..c........Z..b.......i...4....B.. .-2...+3$i.!.......7..._.T..G`...Ar.D.a.....U..^....^.Q.h.._.p.H..9.*O.5)-T_....7}8.>."...j..)e.^..-.'.L....Y9.6d...Z...<.....hygo.z\H.11...q{.*T....V.>K.9\HJ...7.....m.r:.(...s.'5|_...F..X&..>#(..]...H.6.V....(.4z..3,...e.P.r..H..A<a*:&..".7)......$.kv...y.....;....#.b.3.8....dVQ....]*Nk{.....T...o. .../...o5......=.!_..:....L.......S.J..u)\.*....$...I..-...!...kAK%.s.t.*.<.j.......w.z.'.7t....K-:.....:.?....GmA$.j.....@D...q.wE]..').........J.'....
..).p.[..._..6.'Dg.h..,(8:...%_.%....ES_.g.O.Q{.*...=..6{.Tx_.[..d.g...?# .Yk./..Zl.hX.....T....R...z.Y..A/.,......p<G..L'.FN.....O.n...Fz.7Tl}.%0`....].;<.-.$S..#...r.7..7b.0v.>.[...[....S.YNp......C..LN.....z.r.....6.J..".H...%=T.f.O...84........(..r@O#.3C...9.G..m.D.J...a.w....).GuC?.,.].9a..4...1....MoG8l..u..hV.h.6....Z`....+..9.aAW.]..,_7.@...y..._{.....buwy).q.\L.L....E2..~....',.J............Z.._...G......4,....o.w2
...`....qp.. .g..iP.Vdw...W9.B...q..<...F...j..-G.!\..3r+\.T....{d$....Ys..4.J....D.["..-
(E.l..H7.iw.....?....?p..cI#qu...mK.T...qp.[g..%.2...|....7O...u.K..........?....s.......J.#%...;._.....>..Z......7DA...P.fg.......N..Oz..+....3........y..+...r..*.....[...xT...J...}..n...n...V	..P...<..y..U.^.....90.......4..'..p.E..F2.....~.GBG.....@v<....;m	dd..z~..>\..T$.i..Da...M.!xR......x6.h...l...m.I.Zl ..t
.g..c..w...EEtq.s.......8...x.E.|..%e..n..b.FA'..w..
..
.H.d...	...H>K.......O..#'.`....q..0.K>...".c.~.\.......N..$.
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

- https://istio.io/latest/docs/reference/config/security/peer_authentication/#PeerAuthentication
