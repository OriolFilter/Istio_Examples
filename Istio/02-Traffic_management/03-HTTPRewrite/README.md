

# Continues from

- 01-hello_world_1_service_1_deployment

# There were no changes respective to that version

Through rewriting the URI we can point to the root directory from nginx.

```yaml
      rewrite:
        uri: "/"
```

## The idea is that this rewrite is handled "internally" by Istio, not by the Client that started the request


## Practical usages:



If we refactor our application, and for example we previously where hosting an API to the URL `/apiV1` and now it's being hosted in `/api/V1`, we can do the following rule:


```yaml
    - match:
        - uri:
            exact: /apiV1
      route:
        - destination:
            host: mynewapi # the service destination/target
            port:
              number: 80 # whatever port it is
      rewrite:
        uri: "/api/V1"
```

Or if we "upgraded" the API, and the new API (v2) is retro-compatible with the old API (v1), we could do the following to force all the usages from the old API to be handled by the newer version:

```yaml
    - match:
        - uri:
            exact: /api/V1
      route:
        - destination:
            host: mynewapi # the service destination/target
            port:
              number: 80 # whatever port it is
      rewrite:
        uri: "/api/V2"
```
