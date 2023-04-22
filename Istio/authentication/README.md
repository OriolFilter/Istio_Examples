## Authentication

- Between pods

- Between namespaces

- Based on method

- Based on service account(s)

- Custom action (it's in alpha feature, should not focus on it for now)

- Audit / logs



reference (from specific deployment)

https://discuss.istio.io/t/istio-deployment-deny-all-default/10983/6

```yaml
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-reviews"]
```