# Getting Started

The idea of these examples is to get yourself familiarized with the basic elements used on Istio, allowing you to explore the documentation as well of proceeding with other examples or tests on your onw.

On these examples you will find the following Istio resources:

- Gateway
- VirtualService
- DestinationRule

# Examples

- 01-hello_world_1_service_1_deployment

- 02-hello_world_1_service_2_deployments_unmanaged

- ALL NEEDS DOCUMENTATION

- 03-hello_world_1_service_2_deployments_managed_version

- 04-hello_world_1_service_2_deployments_managed_version_defaultnt_namespace

- 05-hello_world_1_Service_Entry





# How to get started?

## Install Istio

Follow [this](https://istio.io/latest/docs/setup/getting-started/) guide to install the `default` profile.

Specifically, the steps of [Download Istio](https://istio.io/latest/docs/setup/getting-started/#download) and [Install Istio][https://istio.io/latest/docs/setup/getting-started/#install). 

Once this is set, proceed with the rest of the installation.

## Setting up a Cluster?

Consider using this.

It's what I used to set up my labs for testing (which is the environment that I to do this repository/set of examples).

https://gitea.filterhome.xyz/ofilter/ansible_kubernetes_cluster

Also, I have added MetalLB to allow for my Load Balancers to get a Local IP and be available through the local network environment.