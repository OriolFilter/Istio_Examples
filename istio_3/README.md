## https://istio.io/latest/docs/examples/microservices-istio/setup-kubernetes-cluster/

### Create namespaces

```shell
export NAMESPACE=tutorial
kubectl create namespace $NAMESPACE
```

### Install istio demo


```shell
istioctl install --set profile=demo
```


### Install telemetry addons

#### Grafana

```shell
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/addons/grafana.yaml
```

#### Prometheus

```shell
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/addons/prometheus.yaml
```

#### Kiali

```shell
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/addons/kiali.yaml
```

#### Jaeger

```shell
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/addons/jaeger.yaml
```

### Create ingress resources

```shell
kubectl apply ./gateway.yaml
```