

# 503 Service Unavailable

Setup:

Ingress (istio-ingress ns) -> Gateway (istio-ingress ns) -> virtual-service (default ns) -> Apache/Nginx SVC (default ns)

Cause of the issue:

```text
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: default
spec:
  gateways:
  - istio-ingress/default
  hosts:
  - '*'
  http:
  - route:
    - destination:
        host: nginx-http
```

There was a typo in the SVC name.

```shell
diff -u -N /tmp/LIVE-456335044/networking.istio.io.v1.VirtualService.default.reviews /tmp/MERGED-1595342442/networking.istio.io.v1.VirtualService.default.reviews
--- /tmp/LIVE-456335044/networking.istio.io.v1.VirtualService.default.reviews   2025-07-19 20:58:22.609372841 +0200
+++ /tmp/MERGED-1595342442/networking.istio.io.v1.VirtualService.default.reviews        2025-07-19 20:58:22.609372841 +0200
@@ -5,7 +5,7 @@
     kubectl.kubernetes.io/last-applied-configuration: |
       {"apiVersion":"networking.istio.io/v1","kind":"VirtualService","metadata":{"annotations":{},"name":"reviews","namespace":"default"},"spec":{"gateways":["istio-ingress/default"],"hosts":["*"],"http":[{"route":[{"destination":{"host":"nginx-http"}}]}]}}
   creationTimestamp: "2025-07-19T18:48:26Z"
-  generation: 3
+  generation: 4
   name: reviews
   namespace: default
   resourceVersion: "5835"
@@ -18,4 +18,4 @@
   http:
   - route:
     - destination:
-        host: nginx-http
+        host: nginx-svc
```

## no healthy upstream

Means that there is no pod available to reply.


The pod is not ready/pods not open.

The SVC is not pointing towards the right backend (check pod's labels, not deployment labels).


## VS not rotating/syntax overlapping

If creating virtual service like the following, the syntax is incorrect, and only will populate the last repeating field (on this scenario would use as a backend apache)

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: simple-vs
  namespace: default
spec:
  gateways:
  - istio-ingress/default
  hosts:
  - '*'
  http:
  - route:
    - destination:
        host: wrong-svc
        host: nginx-svc
        host: apache-svc
```