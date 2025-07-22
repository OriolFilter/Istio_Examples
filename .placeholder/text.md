
whatever is this

https://istio.io/latest/docs/ops/diagnostic-tools/istioctl-analyze/#enabling-validation-messages-for-resource-status

---

https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRedirect

## The idea is that this rewrite is handled "externally" by the client, not by Istio.



## Practical examples


### HTTP to HTTPS redirect.

The following Virtual Service configuration will redirect all the incoming traffic from the gateway `my-gateway` that uses the http protocol, to the https protocol.

In this example, it would forward all the `http` traffic without taking into account which port is used. 

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: to-https-vs
spec:
  hosts:
    - "*"
  gateways:
    - my-gateway
  http:
  - match:
    - name: to_https
      match:
        scheme: http
      redirect:
        scheme: https
```

### Migrated from a domain

The following will update the requests coming "to" the domain `old.domain.com` and rewrite the URL to use the "new" `new.domain.com`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: update-domain-vs
spec:
  hosts:
    - "old.domain.com"
  gateways:
    - helloworld-gateway
  http:
    - name: forward-to-new-domain
      redirect:
        authority: "new.domain.com"
```