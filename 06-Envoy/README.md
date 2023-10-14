
## Description

This section focuses on configuring the object `EnvoyFilter`.

## Examples
 
- 01-Envoy-add-response-headers
- 02-envoy-logging

## Heads up

On the example `02-envoy-logging`, it's a requisite to configure Istio's `meshConfig.accessLogFile` as `/dev/stdout`.

During the installation of the cluster itself, can be set with:

```shell
istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout
```

On the current scenario, I would recommend purging the Istio installation and reinstalling again, as I assume that you
are testing this examples in a sandbox that you are free to "destroy".

### Purging Istio

```shell
istioctl uninstall --purge
```

Then proceed with reinstalling Istio using the command from above.

### What if I don't want to purge Istio?

Modify the IstioOperator similarly as mentioned [here](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/#change-to-the-blocking-by-default-policy), and populate the object with the following fields: 

```yaml
spec:
  profile: minimal
  meshConfig:
    accessLogFile: /dev/stdout
```


## Links of Interest

- https://istio.io/latest/docs/reference/config/networking/envoy-filter/
- https://istio.io/latest/docs/reference/config/networking/envoy-filter/#EnvoyFilter-ApplyTo
- https://github.com/istio/istio/wiki/EnvoyFilter-Samples
- https://istio.io/latest/docs/reference/config/networking/envoy-filter/#EnvoyFilter-Patch-Operation
