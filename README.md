# Disclaimer:

I have absolutely used as a reference and or template other party configurations/files.

I have tried to reference as much as possible as long it's relevant/useful for the reader.

Refer to the specific `README.md` in each example for more information, as the documentation is still in progress.

As per the moment, most of the examples are located in 02-Traffic_management.

Currently, the resources are under a relocation and the folders might contain things that don't _really match the topic_.  

# Stuff

## Directories

```text
├── 00-Troubleshooting
├── 01-Getting_Started
│   ├── 01-hello_world_1_service_1_deployment
│   ├── 02-hello_world_1_service_2_deployments_unmanaged
│   ├── 03-hello_world_1_service_2_deployments_managed_version
│   └── 04-hello_world_1_service_2_deployments_managed_version_foo_namespace
├── 02-Traffic_management
│   ├── 01-2_deployments_method
│   ├── 02-DirectResponse-HTTP-Body
│   ├── 03-HTTPRewrite
│   ├── 04-HTTPRedirect
│   ├── 05a-FaultInjection-delay
│   ├── 05b-FaultInjection-abort
│   ├── 05-hello_world_1_Service_Entry
│   ├── 06-hello_world_1_HTTPS-Service_Entry
│   │   └── src
│   ├── 06-mTLS
│   ├── 07-HTTPS-Gateway-Simple-TLS
│   ├── 08a-HTTPS-min-TLS-version
│   ├── 08b-HTTPS-max-TLS-version
│   ├── 09-HTTPS-backend
│   ├── 10-TCP-FORWARDING
│   ├── 11-TLS-PASSTHROUGH
│   ├── 12-HTTP-to-HTTPS-traffic-redirect
│   └── src
├── 03-Sidecar
│   └── 01-ingress-proxy-forwarding
├── 04-Envoy
│   └── 01-envoy_add_headers
├── 05-MeshConfig
│   └── 01-Outboud-Traffic-Policy
├── 06-AuthorizationPolicy
│   ├── 01-target-namespaces
│   ├── 02-target-service-accounts
│   └── 03-target-deployments
├── 09-Ingress
│   └── 01-Create-Istio-LoadBalancer
├── 10-PeerAuthentication
│   ├── 01-disable-mTLS
│   └── 02-portLevelMtls
├── 99-resources
│   └── HTTPS-NGINX-DOCKERFILE
└── XX-CirtcuitBreaking
```

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

https://istio.io/latest/docs/ops/deployment/deployment-models/

## Services port names

Istio allows to specify which protocol will run through a port.

It requires the name of the port to be set to a specific format `name: <protocol>(-<suffix>)`.

Starting from Kubernetes 1.18, it also can be specified through the `appProtocol` field in the port, resulting in `appProtocol: <protocol>`.

This means that port names should respect this format to avoid issues, and for such be cautious when setting up  the name of the ports. 

This applies to multiple Istio elements, but as well to `kind: Services` from default Kubernetes.

For more information about this behavior, refer to:

https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/#explicit-protocol-selection



# Workload selector is cool

- https://istio.io/latest/docs/reference/config/type/workload-selector/#WorkloadSelector

# Links of interest

- https://istiobyexample.dev/

