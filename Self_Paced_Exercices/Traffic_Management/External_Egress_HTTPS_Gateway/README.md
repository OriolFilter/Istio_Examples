# Prerequisites

This assumes you have `Istio-base` and `IstioD` installed in the cluster.

# Steps

1. Update IstioD to have the setting `meshConfig.OutboundTrafficPolicy` set as `registryOnly`
2. Install an instance of Istio-egress gateway in the namespace `istio-egress`, the service should be created as `clusterIP` instead of `LoadBalancer`.
3. Create a Gateway object that routes the egress traffic for the host `ifconfig.me`, with the port 443 and protocol `HTTPS`. Notice that in the Virtual Service Provided the traffic is routed towards `ifconfig.me` with port 443.

```shell
kubectl create -f ./src
```

```shell
helm showvalues istio/istiod
```

```shell
helm showvalues istio/gateway
```

## Answer

1. Set the OutboundTrafficPolicy to registryOnly
```shell
helm upgrade --reuse-values istiod istio/istiod --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY --wait -n istio-system
```

2.
```shell
helm install istio-egress istio/gateway --set service.type="ClusterIP" -n istio-egress --create-namespace --wait
```

3. Create the Gateway
```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: egress-gateway
  namespace: istio-egress
spec:
  selector:
    app: istio-egress
    app.kubernetes.io/instance: istio-egress
  servers:
    - port:
        number: 443
        name: https
        protocol: https
      hosts:
        - "ifconfig.me"
      tls:
        mode: PASSTHROUGH
```

4. Test

```shell
curl 172.18.10.10
```

```text
83.1.2.3
```


## Links

- https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-OutboundTrafficPolicy-Mode

- https://killercoda.com/lorenzo-g/scenario/egress-gateways