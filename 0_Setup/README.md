# Setting up a Kubernetes Cluster

# How to get started?

## Install Istio

Follow [this](https://istio.io/latest/docs/setup/getting-started/) guide to install the `default` profile.

Specifically, the steps of [Download Istio](https://istio.io/latest/docs/setup/getting-started/#download) and [Install Istio](https://istio.io/latest/docs/setup/getting-started/#install).

Once this is set, proceed with the rest of the installation.

## Setting up a Cluster?

Consider using [this](https://gitea.filterhome.xyz/ofilter/ansible_kubernetes_cluster).

It's what I used to set up my labs for testing (which is the environment that I to do this repository/set of examples).

https://gitea.filterhome.xyz/ofilter/ansible_kubernetes_cluster

Also, I have added MetalLB to allow for my Load Balancers to get a Local IP and be available through the local network environment.

Otherwise, follow the Kind example/guide/instructions.

Note that some examples might be inconsistent through the usage of the **Istioctl** command and helm.