IDK put some text in thQereSQ



### Start the packet capture process

```shell
$ kubectl exec -n default  "$(kubectl get pod -n default -l app1 =helloworld -o jsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port 80  -A
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```


### Logs