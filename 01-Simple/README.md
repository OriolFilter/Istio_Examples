# Simple examples


# Traffic path

## Istio Ingress Controller ---> Gateway -> Virtual Service (-> Destination Route) -> Ingress -> Deployment


# Examples

ALL NEEDS DOCUMENTATION

- 01-hello_world_1_service_1_deployment

- 02-hello_world_1_service_2_deployments_unmanaged

- 03-hello_world_1_service_2_deployments_managed_version

- 04-hello_world_1_service_2_deployments_managed_version_defaultnt_namespace

- 05-hello_world_1_Service_Entry








# TODO

do HTTPS ingress

tcp ingress to minecraft/factorio/zomboid

Service Entry with outbound policy set to `REGISTRY_ONLY`
istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout  --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
(no funca)