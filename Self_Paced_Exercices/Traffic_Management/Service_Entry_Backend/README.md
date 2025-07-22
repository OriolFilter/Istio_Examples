
Create a ServiceEntry that uses as a backend the public endpoint `ifconfig.me`. Use it as an HTTP backend.

## Tests

```shell
curl 172.18.10.10 
```

```text
83.1.2.3 # Made up IP
```

## Links

- https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/#deploy-istio-egress-gateway
- https://artifacthub.io/packages/helm/istio-official/gateway#egress-gateway