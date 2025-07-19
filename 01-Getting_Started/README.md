---
gitea: none
include_toc: true
---

# Getting Started

The idea of these examples is to get yourself familiarized with the basic elements used on Istio, allowing you to
explore the documentation as well of proceeding with other examples or tests on your onw.

On these examples you will find the following Istio resources:

- Gateway
- VirtualService
- DestinationRule

# Examples

- 01-hello_world_1_service_1_deployment

- 02-hello_world_1_service_2_deployments_unmanaged

- 03-hello_world_1_service_2_deployments_managed_version

- 04-hello_world_1_service_2_deployments_managed_version_foo_namespace

## Requirements:

- A Kubernetes cluster (with a CNI network plugin, on my (home) environment I have used [Calico](https://docs.tigera.io/calico/))

- Istio installed