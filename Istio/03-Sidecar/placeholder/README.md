https://github.com/steren/istio.github.io/blob/master/_docs/setup/kubernetes/sidecar-injection.md

https://istio.io/latest/docs/reference/config/networking/sidecar/


# Continues from

- 01-hello_world_1_service_1_deployment



the labbel `workloadSelector` only affects the pods.

```yaml
    workloadSelector:
```


whats this command again?


istioctl operator init


https://istio.io/latest/docs/ops/common-problems/injection/


```sh
kubectl create namespace istio-config
```



No fucking clue on how to make it NOT work.



https://istio.io/latest/blog/2021/discovery-selectors/#discovery-selectors-vs-sidecar-resource



https://istio.io/latest/docs/reference/config/networking/sidecar/

# Sidecar notes

Sidecar describes the configuration of the sidecar proxy that mediates inbound and outbound communication to the
workload instance it is attached to.

By default, Istio will program all sidecar proxies in the mesh with the necessary
configuration required to reach every workload instance in the mesh, as well as accept traffic on all the ports associated
with the workload.

The Sidecar configuration provides a way to fine tune the set of ports, protocols that the proxy will
accept when forwarding traffic to and from the workload. In addition, it is possible to restrict the set of services that
the proxy can reach when forwarding outbound traffic from workload instances.




The behavior of the system is undefined if two or more Sidecar configurations with a workloadSelector select the same workload instance.



https://youtu.be/lnYTqNfyzNk

https://www.youtube.com/watch?v=UJ86BNQEcTA
