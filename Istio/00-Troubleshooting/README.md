IDK put some text in there



### Start the packet capture process on the istio-proxy from a pod.

```shell
$ kubectl exec -n default  "$(kubectl get pod -n default -l app=helloworld -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port 80  -A
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

### Logs

Istio system logs

```shell
kubectl logs -f deployments/istiod -n istio-system
```