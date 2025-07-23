Install the Istio ingress gateway in the `istio-ingress` namespace, with 2 pods instead of a single one.

Set this through the `autoscaling.minReplicas` tag value.

> **Note:\**
> This assumes you have istio/base and istio/istiod installed.


```shell
helm show values istio/gateway | grep -v '^$'
```

```shell
kubectl create namespace istio-ingress
```

```text
namespace/istio-ingress created
```

## Answer

```shell
helm install istio-ingress istio/gateway --version '1.26.2' -n istio-ingress --set autoscaling.minReplicas=2
```

```text
NAME: istio-ingress
LAST DEPLOYED: Wed Jul 23 22:24:05 2025
NAMESPACE: istio-ingress
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
"istio-ingress" successfully installed!

To learn more about the release, try:
  $ helm status istio-ingress -n istio-ingress
  $ helm get all istio-ingress -n istio-ingress

Next steps:
  * Deploy an HTTP Gateway: https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/
  * Deploy an HTTPS Gateway: https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/
```

Initially it will only show a single pod, yet we can see how the HPA resource displays the minPods=2, as we have configured.

Once the istio-ingress reports as correct the HPA replica count will increase to 1, then will proceed to tell the deployment to create a new pod.

```shell
kubectl get pods,deployments,hpa -n istio-ingress
```

```shell
NAME                                 READY   STATUS    RESTARTS   AGE
pod/istio-ingress-68774dc847-5jz8n   1/1     Running   0          2s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-ingress   1/1     1            1           2s

NAME                                                REFERENCE                  TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istio-ingress   Deployment/istio-ingress   cpu: <unknown>/80%   2         5         0          2s
```

After a bit...

```text
NAME                                 READY   STATUS    RESTARTS   AGE
pod/istio-ingress-68774dc847-5jz8n   1/1     Running   0          114s
pod/istio-ingress-68774dc847-z4ljq   1/1     Running   0          108s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-ingress   2/2     2            2           114s

NAME                                                REFERENCE                  TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istio-ingress   Deployment/istio-ingress   cpu: 1%/80%   2         5         2          114s
```