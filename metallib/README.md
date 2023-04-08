[//]: # ()
[//]: # (# https://levelup.gitconnected.com/step-by-step-slow-guide-kubernetes-cluster-on-raspberry-pi-4b-part-3-899fc270600e)

[//]: # ()
[//]: # ()
[//]: # (kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml)

[//]: # (kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml)

[//]: # ()
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"


https://metallb.universe.tf/installation/

https://metallb.universe.tf/configuration/_advanced_l2_configuration/

https://mvallim.github.io/kubernetes-under-the-hood/documentation/kube-metallb.html



```sh
kubectl apply -f - << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.50-192.168.1.130
EOF
```


```sh
kubectl delete -f - << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.50-192.168.1.130
EOF
```



```sh
kubectl apply -f - << EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.50-192.168.1.130
EOF
```




```sh
kubectl delete -f - << EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.50-192.168.1.130
EOF
```


# https://github.com/metallb/metallb/blob/main/design/pool-configuration.md