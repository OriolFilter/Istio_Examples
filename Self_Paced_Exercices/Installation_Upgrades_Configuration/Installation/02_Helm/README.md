# Documentation

https://istio.io/latest/docs/setup/install/helm/

## Deployment Profiles

https://istio.io/latest/docs/setup/additional-setup/config-profiles/#deployment-profiles

## Add Helm chart
```shell
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

```text
"istio" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "metrics-server" chart repository
...Successfully got an update from the "istio" chart repository
Update Complete. ⎈Happy Helming!⎈
```

## Create NS

```shell
kubectl create ns istio-system
```

```text
namespace/istio-system created
```