
## Examples

- 01-ingress-proxy-forwarding

-



Duplicate 01, and show how it also affects traffic between services.00




egress from (pod to pod)

mtls



examples showing application priority (root < namespace < workload)




istioctl install profile=default --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY




```shell
$ kubectl get istiooperators.install.istio.io -n istio-system
NAME              REVISION   STATUS   AGE
installed-state                       8d
```

kubectl patch istiooperators installed-state -n istio-system --patch-file patch.txt


kubectl patch istiooperators installed-state -n istio-system --patch-file patch.yaml  --type merge






---
Set the default behavior of the sidecar for handling outbound traffic from the application. If your application uses one or more external services that are not known apriori, setting the policy to ALLOW_ANY will cause the sidecars to route any unknown traffic originating from the application to its requested destination.



---
https://stackoverflow.com/questions/75093144/istio-sidecar-is-not-restricting-pod-connections-as-desired

https://github.com/istio/istio/issues/33387

https://gist.github.com/GregHanson/3567f5a23bcd58ad1a8acf2a4d1155eb


https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/?_ga=2.259114634.1481027401.1681916557-32589553.1681916557#change-to-the-blocking-by-default-policy







https://docs.tetrate.io/service-bridge/1.6.x/en-us/operations     ?


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