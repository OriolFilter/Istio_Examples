---
gitea: none
include_toc: true
---


# Based on

- [01-hello_world_1_service_1_deployment](../../01-Simple/01-hello_world_1_service_1_deployment)

# Description

On this example, a new Istio Ingress Load Balancer is deployed.

The previous example has been modified to utilize the Ingress resource just deployed.

# Changelog

## Gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: helloworld-gateway
spec:
  selector:
    istio: myingressgateway # use istio default controller
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

The selector `Istio` has been updated to `myingressgateway`, to match the selector of the Istio Ingress Load Balancer that will be created.

## Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: istio-ingress
  labels:
    istio-injection: "enabled"
```

The namespace `istio-ingress` will have the label `istio-injection` with the contents set to `enabled` to allow Istio to automatically inject the Istio sidecars to the resources within that namespace, unless specified otherwise.

## IstioOperator

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: ingress
spec:
  profile: empty # Do not install CRDs or the control plane
  components:
    ingressGateways:
      - name: myistio-ingressgateway
        namespace: istio-ingress
        enabled: true
        label:
          # Set a unique label for the gateway. This is required to ensure Gateways
          # can select this workload
          istio: myingressgateway
  values:
    gateways:
      istio-ingressgateway:
        # Enable gateway injection
        injectionTemplate: gateway
```

The following configuration will create an Istio Ingress Load Balancer named `myistio-ingressgateway`, located at the namespace `istio-ingress`.

The label `istio`, refers to the selector that the `Gateway` resources will use to specify the targeted Istio resource.

# Walkthrough

## Deploy resources

### Create namespace

```shell
kubectl apply -f 01-namespace.yaml
```
```text
namespace/istio-ingress created
```

### Create / Install the Istio Ingress resource


```shell
istioctl install -f ingress.yaml
```
```text
This will install the Istio 1.17.2 empty profile into the cluster. Proceed? (y/N) y
✔ Ingress gateways installed                                                                                                                                                                                                          
✔ Installation complete                                                                                                                                                                                                               
Thank you for installing Istio 1.17.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/hMHGiwZHPU7UQRWe9
```

### Deploy gateway

```shell
kubectl apply -f gateway.yaml
```
```text
 
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld-vs created
```

### Deploy deployment

```shell
kubectl apply -f deployment.yaml
```
```text
service/helloworld created
deployment.apps/helloworld-nginx created
```

## Testing deployment

### Get Load Balancer IP

```shell
kubectl get svc -n istio-ingress
```
```text
NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                                      AGE
myistio-ingressgateway   LoadBalancer   10.102.158.128   192.168.1.51   15021:31181/TCP,80:30090/TCP,443:31285/TCP   5m10s
```

### Curl

The request results in status code `200`, meaning a correct handling of the request.

```shell
curl 192.168.1.51/helloworld -I
```
```text
HTTP/1.1 200 OK
server: istio-envoy
date: Sun, 23 Apr 2023 06:40:57 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 28 Mar 2023 15:01:54 GMT
etag: "64230162-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 15
```
# Cleanup

[Yeah no idea, gl with that.](https://stackoverflow.com/a/55731730)

```shell
kubectl delete -f ./deployment.yaml
kubectl delete -f ./gateway.yaml
```
```text
service "helloworld" deleted
deployment.apps "helloworld-nginx" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
```

```shell
istioctl uninstall --purge
```

Also read that "just removing" the namespace works to purge the config/remove resources.

Meanwhile, I did that (and seems like it performed correctly), I am not entirely sure about it. I'm not bothered myself as the environment where I am performing the tests is intended to be destroyed anytime and recreated, yet in a production environment I am not sure how this would need to be approached.

Maybe with a `kubectl get all -A` and through `grep` and `less` find resources and configurations, and delete them manually.

```shell
kubectl delete namespace istio-ingress
```

# Troubleshooting

## curl: (7) Failed to connect to 192.168.1.51 port 80 after 2 ms: Couldn't connect to server

Ensure that the gateway is using the correct `selector` to target the Istio Ingress Load Balancer created.  

# Links of interest

- https://istio.io/latest/docs/setup/additional-setup/gateway/#deploying-a-gateway