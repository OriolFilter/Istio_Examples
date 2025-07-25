# Disclaimer:

I have absolutely used as a reference and/or template other parties configurations/files as well of documentations and examples.

I have tried to reference as much as possible as long it's relevant/useful for the reader.

Refer to the specific `README.md` in each example for more information.

# Tree of folders

```shell
tree -d | grep -v src$
```

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
│   ├── 02-Outboud-Traffic-Policy
│   ├── 03-HTTPS-backend
├── 05-Sidecar
│   ├── 01-ingress-proxy-forwarding
│   └── 02-egress-proxy
├── 06-Envoy
│   ├── 01-Envoy-add-response-headers
│   └── 02-envoy-logging
├── 08-AuthorizationPolicy
│   ├── 01-AuthorizationPolicy-Target-Namespaces
│   ├── 02-AuthorizationPolicy-Target-Service-Accounts
│   └── 03-AuthorizationPolicy-Target-Deployments
├── 09-Ingress
│   └── 01-Ingress-IstioOperator
│       └── IstioOperator
├── 10-mTLS_PeerAuthentication
│   ├── 01-mTLS
│   ├── 02-disable-mTLS
│   └── 03-mTLS-per-port-settings
├── 11-Fault_Injection
│   ├── 01-FaultInjection-delay
│   └── 02-FaultInjection-abort
├── 12-CircuitBreaking
├── 13-monitoring
│   ├── 01-Create_Prometheus_Stack
│   ├── 02-Add_Istio_Scrapping_Metrics
│   └── 03-Grafana_Istio_Dashboards
├── 90-MixConfigs
│   ├── 01-HTTPS-Gateway_Service_Entry
│   └── Minecraft
│       └── Istio-Ingress
└── 99-resources
    └── HTTPS-NGINX-DOCKERFILE
```

#### "Why is 07 missing"

Previously there was a folder that got refactored.

Eventually the spot will be filled back.

Want to avoid renaming folders unless required as it could break link references within the documentation.

# Setting up a cluster to use this exercises as reference.

Refer to the directory [Setup](./Setup).

Contains instructions for setting up a Kind cluster, and also a script of personal use I used a while ago for a bland Kubernetes/kubeadm cluster.

Kind is much more recent, so a couple of things might differ when using that stack, but bases and concepts remain the same. 

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
