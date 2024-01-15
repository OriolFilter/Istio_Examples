## Description

Through the use of Prometheus CRDs, we deploy a PodMonitor and ServiceMonitor objects, which will scrap metrics from the Envoy Proxies attached to each pod and Istiod deployment. 

## Requirements

- Complete step [01-Create_Prometheus_Stack](../01-Create_Prometheus_Stack)

## Istio Metrics

Now that a functional Prometheus-Grafana-Alert manager set up.

The next step is to deploy scrapping Prometheus jobs/configs to gather:

- Envoy proxy metrics
- Istiod metrics.

> **Note**: \
> That the operators deployed are based off the [Istio Prometheus Operator Example](https://github.com/istio/istio/blob/1.20.2/samples/addons/extras/prometheus-operator.yaml)

```shell
kubectl create -f PrometheusIstioAgent.yaml
```

```text
servicemonitor.monitoring.coreos.com/istiod-metrics-monitor created
podmonitor.monitoring.coreos.com/envoy-stats-monitor created
```

To update the list of Prometheus targets, we can wait for a bit until it gets picked up automatically, idk give it a minute or two, get off the PC and grab some whatever or stretch your legs.

### Check Targets

Once the Prometheus pod is up and running again, if we access the website service, and access to the section **Status > Targets**, we can list all the available Targets.

Once there, I am able to see the following entries:

- **podMonitor/observability/envoy-stats-monitor/0 (15/15 up)**

- **serviceMonitor/observability/istiod-metrics-monitor/0 (2/2 up)**

### Check through Prometheus queries

Now, back to the **Graph** section, we can confirm if we are receiving metrics from **Istiod** and **Envoy**.

#### Istiod

Very simple and straightforward, the uptime for each one of the **Istiod** pods.

```promql
istiod_uptime_seconds
```

#### Envoy

Requests grouped by `destination_service_name`.

```promql
sum(istio_requests_total) by (destination_service_name)
```