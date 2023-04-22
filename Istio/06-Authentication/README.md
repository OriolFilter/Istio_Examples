## Authentication

- Based on deployments

- Based on namespaces (done)

- Based on method (somewhat done, so I will mark it as valid)

- Based on service account(s)

- Custom action (it's in alpha feature, should not focus on it for now)

- Audit / logs (shold be the 5th)



reference (from specific deployment)

https://discuss.istio.io/t/istio-deployment-deny-all-default/10983/6

```yaml
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-reviews"]
```


JWT seems important, refer to source.requestPrincipals