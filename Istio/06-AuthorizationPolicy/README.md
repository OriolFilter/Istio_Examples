## Authentication

- Based on namespaces (done)
  
- Based on method (somewhat done, so I will mark it as valid)

- Based on service account(s) (somewhat done)

- Custom action (it's in alpha feature, should not focus on it for now)

- Audit / logs (should be the 3th)

- disable mTLS (4th)

JWT seems important, refer to source.requestPrincipals

https://istio.io/latest/docs/tasks/security/authentication/



Per deployment:
```yaml
  selector:
    matchLabels:
      app: myapi
```