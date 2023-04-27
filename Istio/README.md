# Disclaimer:

I have absolutely used as a reference and or template other party configurations/files.

I have tried to reference as much as possible as long it's relevant/useful for the reader.

# Stuff

## Glossary

https://istio.io/latest/docs/reference/glossary/


## Workload

https://istio.io/latest/docs/reference/glossary/#workload

https://kiali.io/docs/architecture/terminology/concepts/#workload


https://istio.io/latest/docs/ops/deployment/vm-architecture/


## Sidecar

https://kubebyexample.com/learning-paths/istio/intro


# Notes for myself

Internal and external authentication should be set together.


https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/


## Services port names

Istio allows to specify which protocol will run through a port.

It requires the name of the port to be set to a specific format `name: <protocol>(-<suffix>)`.

Starting from Kubernetes 1.18, it also can be specified through the `appProtocol` field in the port, resulting in `appProtocol: <protocol>`.

This means that port names should respect this format to avoid issues, and for such be cautious when setting up  the name of the ports. 

This applies to multiple Istio elements, but as well to `kind: Services` from default Kubernetes.

For more information about this behavior, refer to:

https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/#explicit-protocol-selection


# Links of interest

- https://istiobyexample.dev/

