
# Description

Expecting this to be done in the Istio v1.26.

Will try to limit myself within the realms of the official documentation, or whatever I can do from the command shell.

## Links

### https://training.linuxfoundation.org/istio-certified-associate-ica-program-changes/

#### topics

Installation, Upgrades, and Configuration – 20%

    Installing Istio with istioctl or Helm
    Installing Istio in Sidecar or Ambient Mode
    Customizing your Istio Installation
    Upgrading Istio (Canary, In-Place)

Traffic Management – 35%

    Configuring Ingress and Egress Traffic
    Configuring Routing within a Service Mesh
    Defining Traffic Policies with Destination Rules
    Configuring Traffic Shifting
    Connecting In-Mesh Workloads to External Workloads and Services
    Using Resilience Features (circuit breaking, failover, outlier detection, timeouts, retries)
    Using Fault Injection

Securing Workloads – 25%

    Configuring Authorization
    Configuring Authentication (mTLS, JWT)
    Securing Edge Traffic with TLS

Troubleshooting – 20%

    Troubleshooting Configuration
    Troubleshooting the Mesh Control Plane
    Troubleshooting the Mesh Data Plane

# Init

## Create cluster

```shell
cat << EOF > /tmp/kind-conf.yaml 
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
EOF
```

```shell
kind create cluster --name istio-testing --config /tmp/kind-conf.yaml
```
```text
Creating cluster "istio-testing" ...
 ✓ Ensuring node image (kindest/node:v1.33.1) 🖼
 ✓ Preparing nodes 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-istio-testing"
You can now use your cluster with:

kubectl cluster-info --context kind-istio-testing

Thanks for using kind! 😊
```

```shell
kubectl cluster-info --context kind-istio-testing
```

```text
Kubernetes control plane is running at https://127.0.0.1:34753
CoreDNS is running at https://127.0.0.1:34753/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Set context
```shell
kubectl config use-context kind-istio-testing
```


> https://istio.io/latest/docs/setup/platform-setup/kind/

## Install Metrics Server

```shell
helm install metrics-server --namespace kube-system metrics-server/metrics-server --wait --set "args[0]"="--kubelet-insecure-tls"
```

```text
NAME: metrics-server
LAST DEPLOYED: Sat Jul 19 18:50:12 2025
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
***********************************************************************
* Metrics Server                                                      *
***********************************************************************
  Chart version: 3.12.2
  App version:   0.7.2
  Image tag:     registry.k8s.io/metrics-server/metrics-server:v0.7.2
***********************************************************************
```

## Install MetalLB

Test connectivity towards the nodes with a ping.

```text
➜  ~ kubectl get nodes -owide
NAME                          STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
istio-testing-control-plane   Ready    control-plane   44h   v1.33.1   172.18.0.3    <none>        Debian GNU/Linux 12 (bookworm)   6.15.6-arch1-1   containerd://2.1.1
istio-testing-worker          Ready    <none>          44h   v1.33.1   172.18.0.2    <none>        Debian GNU/Linux 12 (bookworm)   6.15.6-arch1-1   containerd://2.1.1
➜  ~ ping 172.18.0.3
PING 172.18.0.3 (172.18.0.3) 56(84) bytes of data.
64 bytes from 172.18.0.3: icmp_seq=1 ttl=64 time=0.041 ms
64 bytes from 172.18.0.3: icmp_seq=2 ttl=64 time=0.027 ms
^C
--- 172.18.0.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1043ms
rtt min/avg/max/mdev = 0.027/0.034/0.041/0.007 ms
➜  ~ ping 172.18.0.2
PING 172.18.0.2 (172.18.0.2) 56(84) bytes of data.
64 bytes from 172.18.0.2: icmp_seq=1 ttl=64 time=0.043 ms
64 bytes from 172.18.0.2: icmp_seq=2 ttl=64 time=0.030 ms
^C
--- 172.18.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.030/0.036/0.043/0.006 ms
```

Get the range for the Kind nodes.

```shell
docker inspect kind | jq ".[].IPAM.Config"
```

```text
[
  {
    "Subnet": "172.18.0.0/16",
    "Gateway": "172.18.0.1"
  },
  {
    "Subnet": "1234:asdb:asd:asd1::/64",
    "Gateway": "asd2:asda:dds:adaa::1"
  }
]
```

```shell
helm repo add metallb https://metallb.github.io/metallb
```

```text
"metallb" has been added to your repositories
```

Create the namespace for metallb

```shell
kubectl create ns metallb
```

```text
namespace/metallb created
```

```shell
helm install metallb metallb/metallb --namespace metallb
```

```text
I0719 19:07:58.940256  578922 warnings.go:110] "Warning: spec.template.spec.containers[3].ports[0]: duplicate port name \"monitoring\" with spec.template.spec.containers[0].ports[0], services and probes that select ports by name will use spec.template.spec.containers[0].ports[0]"
NAME: metallb
LAST DEPLOYED: Sat Jul 19 19:07:58 2025
NAMESPACE: metallb
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
MetalLB is now running in the cluster.

Now you can configure it via its CRs. Please refer to the metallb official docs
on how to use the CRs.
```

⚠️⚠️⚠️ **UPDATE THE ADDRESSES RANGE TO MATCH THE DOCKER NETWORK RANGE** ⚠️⚠️⚠️

```shell
kubectl create -f - << EOF
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: main-pool
  namespace: metallb
spec:
  addresses:
  - 172.18.10.10-172.18.10.20  
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: main-advertising
  namespace: metallb
spec:
  ipAddressPools:
  - main-pool
EOF
```

```text
ipaddresspool.metallb.io/main-pool created
l2advertisement.metallb.io/main-pool created
```

# Killer coda Exercises (1.18)

https://killercoda.com/ica

Examples might be EXTREMELY OUTDATED (we're talking about the version 1.18 while the exam will be in the 1.26)

# Killer coda (1.24)

https://github.com/killercoda/scenarios-istio?tab=readme-ov-file

# Other things to take a look (eventually idk i havent yet no clue if its good nor relevant)

https://medium.com/@wattsdave/istio-certified-associate-ica-exam-prep-51b59bdd372f

https://www.cncf.io/training/certification/ica/

