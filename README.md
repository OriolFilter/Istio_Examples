# Disclaimer:

I have absolutely used as a reference and/or template other parties configurations/files as well of documentations and examples.

I have tried to reference as much as possible as long it's relevant/useful for the reader.

Refer to the specific `README.md` in each example for more information, as the documentation is still in progress.

Currently, the resources are under a relocation and the folders might contain things that don't _really match the topic_.  


# Glossary

https://istio.io/latest/docs/reference/glossary/




## Services port names

Istio allows to specify which protocol will run through a port.

It requires the name of the port to be set to a specific format `name: <protocol>(-<suffix>)`.

Starting from Kubernetes 1.18, it also can be specified through the `appProtocol` field in the port, resulting in `appProtocol: <protocol>`.

This means that port names should respect this format to avoid issues, and for such be cautious when setting up  the name of the ports. 

This applies to multiple Istio elements, but as well to `kind: Services` from default Kubernetes.

For more information about this behavior, refer to:

https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/#explicit-protocol-selection



# Links of interest

- https://istio.io/latest/docs/

- https://istiobyexample.dev/

- https://www.istioworkshop.io/

- https://istio.io/latest/news/

- https://istio.io/latest/blog/

- https://istio.io/latest/about/ecosystem/