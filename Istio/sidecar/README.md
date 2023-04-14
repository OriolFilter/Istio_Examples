https://istio.io/latest/docs/reference/config/networking/sidecar/


https://istio.io/latest/docs/reference/glossary/#workload


I am not very sure on how or why to use this...



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