# Disclaimer:

I have absolutely used as a reference and/or template other parties configurations/files as well of documentations and examples.

I have tried to reference as much as possible as long it's relevant/useful for the reader.

Refer to the specific `README.md` in each example for more information, as (**my**) documentation is still in progress.

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

# Tree of folders

```text
├── 00-Troubleshooting
├── 01-Getting_Started
│   ├── 01-hello_world_1_service_1_deployment
│   ├── 02-hello_world_1_service_2_deployments_unmanaged
│   ├── 03-hello_world_1_service_2_deployments_managed_version
│   └── 04-hello_world_1_service_2_deployments_managed_version_foo_namespace
├── 02-Traffic_management
│   ├── 01-header_routing
│   ├── 02-DirectResponse-HTTP-Body
│   ├── 03-HTTPRewrite
│   └── 04-HTTPRedirect
├── 03-Gateway_Ingress
│   ├── 01-Host_Based_Routing
│   ├── 02-Restrict_Namespaces
│   ├── 03-HTTPS-Gateway-Simple-TLS
│   ├── 04a-HTTPS-min-TLS-version
│   ├── 04b-HTTPS-max-TLS-version
│   ├── 05-TCP-FORWARDING
│   ├── 06-TLS-PASSTHROUGH
│   └── 07-HTTP-to-HTTPS-traffic-redirect
├── 04-Backends
│   ├── 01-Service_Entry
│   └── 02-HTTPS-backend
├── 05-Sidecar
│   ├── 01-ingress-proxy-forwarding
│   └── 02-egress-proxy
├── 08-AuthorizationPolicy
│   ├── 01-target-namespaces
│   ├── 02-target-service-accounts
│   └── 03-target-deployments
├── 09-Ingress
│   └── 01-Create-Istio-LoadBalancer
├── 10-mTLS_PeerAuthentication
│   ├── 01-disable-mTLS
│   ├── 02-portLevelMtls
│   └── 06-mTLS
├── 11-Fault_Injection
│   ├── 05a-FaultInjection-delay
│   └── 05b-FaultInjection-abort
├── 12-CircuitBreaking
├── 90-MixConfigs
│   ├── 01-HTTPS-Gateway_Service_Entry
│   └── Minecraft
└── 99-resources
    └── HTTPS-NGINX-DOCKERFILE
```