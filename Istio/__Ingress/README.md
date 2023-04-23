https://istio.io/latest/docs/tasks/traffic-management/ingress/


TLS
https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/




https://istio.io/latest/docs/setup/additional-setup/gateway/#deploying-a-gateway


kubectl apply -f 01-namespace.yaml

istioctl install -f ingress.yaml


kubectl get all -A | grep myistio
istio-ingress    pod/myistio-ingressgateway-5cdcd89cfb-s4fsz       1/1     Running   0              43s
istio-ingress   service/myistio-ingressgateway   LoadBalancer   10.102.38.206    192.168.1.51   15021:30287/TCP,80:30979/TCP,443:31405/TCP   43s
istio-ingress    deployment.apps/myistio-ingressgateway    1/1     1            1           44s
istio-ingress    replicaset.apps/myistio-ingressgateway-5cdcd89cfb   1         1         1       44s
istio-ingress   horizontalpodautoscaler.autoscaling/myistio-ingressgateway   Deployment/myistio-ingressgateway   <unknown>/80%   1         5         1          44s


---

It gets its own service account.

We can use this to restrict the network activity and enforce traffic rules.

```shell
kubectl get pod -n istio-ingress myistio-ingressgateway-5cdcd89cfb-s4fsz -o jsonpath='{.spec.serviceAccount}' 
```
```text
myistio-ingressgateway-service-account
```
