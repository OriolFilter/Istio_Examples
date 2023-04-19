
## Examples

- 01-ingress-proxy-forwarding

-







egress from (pod to pod)

mtls





---

https://istio.io/latest/docs/reference/config/networking/sidecar/


https://istio.io/latest/docs/reference/glossary/#workload


I am not very sure on how or why to use this...



NOT HOW TO TRIGGER / UNTRIGGER IT

```yaml
apiVersion:
 networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: foo
spec:
  egress:
    - hosts:
      - "./*"
      - "istio-system/*"
```



whats this again??

istio operator right? ye, but what is it again? I think I checked this time ago when doing something about creating a new ingress


kubectl get io -A


2023-04-17T00:08:00.086475Z     info    validationController    Not ready to switch validation to fail-closed: dummy invalid config not rejected


2023-04-17T00:08:04.012630Z     info    validationServer        configuration is invalid: gateway must have at least one server




kubectl logs -f deployments/istiod -n istio-system   

https://istio.io/latest/docs/reference/config/networking/sidecar/




  egress:
  - port:
      number: 8080
      protocol: HTTP
    hosts:
    - "staging/*"



With the YAML above, the sidecar proxies the traffic that’s bound for port 8080 for services running in the staging namespace.








- Confirm pod ingress port forwarding

- Confirm it can reach other places / namespaces / resources (pod egress)

- mtls (somehow)


# Ingress

Does stuff

# Egress

What is "bind"

# CaptureMode

Not my problem rn