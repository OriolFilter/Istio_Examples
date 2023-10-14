## Description

On these examples, a `Sidecar` will be configured.

## Examples

- 01-ingress-proxy-forwarding
- 02-egress-proxy

## Heads up

On the example `02-egress-proxy`, it's a requisite to configure Istio's `meshConfig.outboundTrafficPolicy.mode` as "REGISTRY_ONLY".

During the installation of the cluster itself, can be set with.

```shell
istioctl install profile=default --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
```

On the current scenario, I would recommend purging the Istio installation and reinstalling again, as I assume that you 
are testing this examples in a sandbox that you are free to "destroy".

### Purging Istio

```shell
istioctl uninstall --purge
```

Then proceed with reinstalling Istio using the command from above.

### What if I don't want to purge Istio?

Modify the IstioOperator as mentioned [here](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/#change-to-the-blocking-by-default-policy).