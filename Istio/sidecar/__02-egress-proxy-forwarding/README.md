# Continues from

- 01-ingress-proxy-forwarding

# Description

This example configures the sidecar proxy on the pods created, to forward the traffic ongoing (egress)

- Configure egress to a different namespace?


> the configured meshconfig.rootNamespace namespace (istio-system by default)
https://istio.io/latest/docs/ops/best-practices/traffic-management/#cross-namespace-configuration




CANT MAKE IT WORK CANT MAKE IT WORK CANT MAKE IT WORK






istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout  --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY






---

kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}'

kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- /bin/bash

kubectl exec -i -t "$(kubectl get pod -l app=helloworld | tail -n 1 | awk '{print $1}')" -- curl internal.foo.svc.cluster.local


curl helloworld.default.svc.cluster.local


curl internal.foo.svc.cluster.local
curl: (6) Could not resolve host: internal.foo.svc.cluster.local


helloworld.default.svc.cluster.local:8080


 kubectl exec -i -n foo -t "$(kubectl get pod -l app=internal -n foo | tail -n 1 | awk '{print $1}')" -- /bin/bash