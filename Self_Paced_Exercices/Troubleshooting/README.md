
# Enable Access logs for Envoy Proxy

Enable the usage of sidecar containers with the `istio-injection` label in the namespace.

`istio-injection=enabled`

Either use the istioctl command line to enable the logs.

Or

Update the helm install to have them.

--set meshConfig.accessLogFile=/dev/stdout

```shell
helm upgrade --reuse-values --set meshConfig.accessLogFile="/dev/stdout" -n istio-system istiod istio/istiod 
```

This modifies the CM from Istio, adding the accessLogFile.

```shell
kubectl get cm -n istio-system istio -oyaml    
```

```text
                 
apiVersion: v1
data:
  mesh: |-
    accessLogFile: /dev/stdout
    defaultConfig:
      discoveryAddress: istiod.istio-system.svc:15012
    defaultProviders:
      metrics:
```

Without a need of rebooting the sidecards, we can already visualize the logs.

```text
[2025-07-19T23:40:50.637Z] "- - -" 0 - - - "-" 1419 2816 6 - "-" "-" "-" "-" "10.244.1.20:443" inbound|443|| 127.0.0.6:47875 10.244.1.20:443 10.244.1.20:49964 outbound_.443_._.helloworld.default.svc.cluster.local -
[2025-07-19T23:40:51.652Z] "- - -" 0 - - - "-" 859 2256 6 - "-" "-" "-" "-" "10.244.1.20:443" outbound|443||helloworld.default.svc.cluster.local 10.244.1.20:49966 10.96.223.123:443 10.244.1.20:44398 - -
[2025-07-19T23:40:51.653Z] "- - -" 0 - - - "-" 1419 2816 6 - "-" "-" "-" "-" "10.244.1.20:443" inbound|443|| 127.0.0.6:55509 10.244.1.20:443 10.244.1.20:49966 outbound_.443_._.helloworld.default.svc.cluster.local -
```


# rbac_access_denied_matched_policy

This implies that an Authorization policy has denied the access.

```shell

```


