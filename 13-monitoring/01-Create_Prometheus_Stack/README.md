---
gitea: none
include_toc: true
---

# Description

This example deploys a Prometheus stack (Prometheus, Grafana, Alert Manager) through helm.

This will be used as a base for the future examples.

It's heavily recommended to have a base knowledge of Istio before proceeding to modify the settings according to your needs.

## Requisites

- Istio deployed and running at the namespace `istio-system`.
- Helm installed.

# Istio Files

## Gateway

Simple HTTP gateway.

It only allows traffic from the domain `my.home`, and it's subdomains.

Listens to the port 80 and expects HTTP (unencrypted) requests.

> **Note:**
> I assume the Gateway is already deployed, therefore on the walkthrough it's not mentioned nor specified. If you don't have a gateway, proceed to deploy one before continuing. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: local-gateway
  namespace: default
spec:
  selector:
    istio: local-ingress
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "my.home"
        - "*.filter.home"
```

## VirtualService.yaml

2 simple Virtual Services for the Grafana and Prometheus services/dashboards.

URL for each one are:

- prometheus.my.home

- grafana.my.home

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana-vs
  namespace: default
  labels:
    app: grafana
spec:
  hosts:
    - "grafana.my.home"
  gateways:
    - default/local-gateway
  http:
    - route:
        - destination:
            host: prometheus-stack-01-grafana.observability.svc.cluster.local
            port:
              number: 80
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: prometheus-vs
  namespace: observability
  labels:
    app: prometheus
spec:
  hosts:
    - "prometheus.my.home"
  gateways:
    - default/local-gateway
  http:
    - route:
        - destination:
            host: prometheus-stack-01-kube-p-prometheus.observability.svc.cluster.local
            port:
              number: 9090
```

# Walkthrough

## Create Observability NS

```shell
kubectl create namespace
```

Placeholder namespace annotation, **istio-injection** will be enabled after the installation is completed.

If istio-injection is enabled, Helm installation will **fail**.

I have to check on what/why.

```shell
kubectl label namespaces observability istio-injection=disabled --overwrite=true
```

# PersistentVolume

I'm using a NFS provisioner, you can use whatever you want. (Optional)

On the file `stack_values.yaml` I specified that 2 volumes will be provisioned, one for Prometheus, and another one for AlertManager.

If you don't want to provision volumes, set that file to blank, or on the installation step, remove the line that specifies such line.

As well increased the retention from 10 days (default value), to 30 days, but since you won't have a volume, don't think that will be much of an issue for you...

## Installation

I will be installing Prometheus Operator through Helm.

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```text
"prometheus-community" has been added to your repositories
```

```shell
helm show values prometheus-community/kube-prometheus-stack
```

```text
A lot of text, recommended to save the output on a file and you go through it (at latest use control+f or whatever other search option to find the things you might be interested on replacing/changing)
```

My stack_Values.yaml file is:

```yaml
prometheus:
  prometheusSpec:
    retention: "30d"
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: slow-nfs-01
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 50Gi
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: slow-nfs-01
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 10Gi
```

Besides the volumes mentioned in [here](#persistentvolume), increased the retention from 10 days to 30.

If you haven't configured a PersistentVolume storage, just skip the `--set` lines referencing such. Note that once the pod is restarted, all data will be lost.

```shell
helm install prometheus-stack-01 prometheus-community/kube-prometheus-stack \
  -n observability \
  --values ./src/stack_values.yaml
```

```text
NAME: prometheus-stack-01
LAST DEPLOYED: Sun Jan 14 22:34:11 2024
NAMESPACE: observability
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace observability get pods -l "release=prometheus-stack-01"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

### Check running pods in namespace

Everything seems to be deployed and working correctly.

```shell
kubectl get pods -n observability
```

```text
NAME                                                      READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-stack-01-kube-p-alertmanager-0    2/2     Running   0          73s
prometheus-prometheus-stack-01-kube-p-prometheus-0        2/2     Running   0          73s
prometheus-stack-01-grafana-69bd95649b-w67xg              3/3     Running   0          76s
prometheus-stack-01-kube-p-operator-b97d5f9cc-cm2pl       1/1     Running   0          76s
prometheus-stack-01-kube-state-metrics-554fd7bf8b-z62gv   1/1     Running   0          76s
prometheus-stack-01-prometheus-node-exporter-7bwbd        1/1     Running   0          76s
prometheus-stack-01-prometheus-node-exporter-dvqc6        1/1     Running   0          76s
prometheus-stack-01-prometheus-node-exporter-nfm5g        1/1     Running   0          76s
prometheus-stack-01-prometheus-node-exporter-ssfkb        1/1     Running   0          76s
```

### Enable Istio Injection

Let's enable back istio-injection on the namespace.

```shell
kubectl label namespaces observability istio-injection=enabled --overwrite=true
```

### Delete all pods so are recreated with the istio sidecar

To update the containers we will need to delete/recreate all of them.

```shell
kubectl delete pods -n observability --all
```

```text
pod "alertmanager-prometheus-stack-01-kube-p-alertmanager-0" deleted
pod "prometheus-prometheus-stack-01-kube-p-prometheus-0" deleted
pod "prometheus-stack-01-grafana-69bd95649b-w67xg" deleted
pod "prometheus-stack-01-kube-p-operator-b97d5f9cc-cm2pl" deleted
pod "prometheus-stack-01-kube-state-metrics-554fd7bf8b-z62gv" deleted
pod "prometheus-stack-01-prometheus-node-exporter-7bwbd" deleted
pod "prometheus-stack-01-prometheus-node-exporter-dvqc6" deleted
pod "prometheus-stack-01-prometheus-node-exporter-nfm5g" deleted
pod "prometheus-stack-01-prometheus-node-exporter-ssfkb" deleted
```

### Check pods status (again)

Everything seems to be deployed and working correctly.

```shell
kubectl get pods -n observability
```

```text
NAME                                                      READY   STATUS    RESTARTS      AGE
alertmanager-prometheus-stack-01-kube-p-alertmanager-0    3/3     Running   0             44s
prometheus-prometheus-stack-01-kube-p-prometheus-0        3/3     Running   0             43s
prometheus-stack-01-grafana-69bd95649b-24v58              4/4     Running   0             46s
prometheus-stack-01-kube-p-operator-b97d5f9cc-5bdwh       2/2     Running   1 (43s ago)   46s
prometheus-stack-01-kube-state-metrics-554fd7bf8b-wjw4d   2/2     Running   2 (41s ago)   46s
prometheus-stack-01-prometheus-node-exporter-4266g        1/1     Running   0             46s
prometheus-stack-01-prometheus-node-exporter-lmxdj        1/1     Running   0             45s
prometheus-stack-01-prometheus-node-exporter-shd72        1/1     Running   0             45s
prometheus-stack-01-prometheus-node-exporter-wjhdr        1/1     Running   0             45s
```

### Gateway

I have my gateways already created (on this scenario I will be using the local gateway).

### VirtualService

I will create 2 Virtual Service entries, one for the Grafana dashboard, and another for the Prometheus dashboard:

- Prometheus dashboard URL: "prometheus.llb.filter.home"
- Grafana dashboard URL: "grafana.llb.filter.home"

```text
kubectl apply -f ./src/VirtualService.yaml
```

```shell
virtualservice.networking.istio.io/grafana-vs created
virtualservice.networking.istio.io/prometheus-vs created
```

## Prometheus

As a simple example of being able to access kubernetes metrics, you can run the following promql queries:

### Running pods per node

We can see the value "node=XXXX", which matches the node from our Kubernetes nodes available within the cluster.

```promql
kubelet_running_pods
```

### Running pods per namespace

Right now, on the namespace "observability" I have a total of 9 pods running.

```promql
sum(kube_pod_status_ready) by (namespace)
```

You can verify this by running:

```shell
kubectl get pods -n observability --no-headers=true | nl
```

```text
     1  alertmanager-prometheus-stack-01-kube-p-alertmanager-0    3/3   Running   0             40m
     2  prometheus-prometheus-stack-01-kube-p-prometheus-0        3/3   Running   0             40m
     3  prometheus-stack-01-grafana-69bd95649b-24v58              4/4   Running   0             40m
     4  prometheus-stack-01-kube-p-operator-b97d5f9cc-5bdwh       2/2   Running   1 (40m ago)   40m
     5  prometheus-stack-01-kube-state-metrics-554fd7bf8b-wjw4d   2/2   Running   2 (40m ago)   40m
     6  prometheus-stack-01-prometheus-node-exporter-4266g        1/1   Running   0             40m
     7  prometheus-stack-01-prometheus-node-exporter-lmxdj        1/1   Running   0             40m
     8  prometheus-stack-01-prometheus-node-exporter-shd72        1/1   Running   0             40m
     9  prometheus-stack-01-prometheus-node-exporter-wjhdr        1/1   Running   0             40m
```

Which returns a total of 9 pods, with s status "running"

### Running containers per namespace

Currently, this is returning 18 containers running on the namespace **observability**.

```promql
sum(kube_pod_container_status_running) by (namespace)
```

Very much, listing again the pods running within the namespace, and just counting the values, I can confirm the total of containers running within the namespace, totals up to 18, matching the prometheus data.

```shell
kubectl get pods -n observability
```

```text
NAME                                                      READY   STATUS    RESTARTS      AGE
alertmanager-prometheus-stack-01-kube-p-alertmanager-0    3/3     Running   0             45m
prometheus-prometheus-stack-01-kube-p-prometheus-0        3/3     Running   0             45m
prometheus-stack-01-grafana-69bd95649b-24v58              4/4     Running   0             45m
prometheus-stack-01-kube-p-operator-b97d5f9cc-5bdwh       2/2     Running   1 (45m ago)   45m
prometheus-stack-01-kube-state-metrics-554fd7bf8b-wjw4d   2/2     Running   2 (45m ago)   45m
prometheus-stack-01-prometheus-node-exporter-4266g        1/1     Running   0             45m
prometheus-stack-01-prometheus-node-exporter-lmxdj        1/1     Running   0             45m
prometheus-stack-01-prometheus-node-exporter-shd72        1/1     Running   0             45m
prometheus-stack-01-prometheus-node-exporter-wjhdr        1/1     Running   0             45m
```
