## Description

Shares some dashboards ready to use once Istio metrics are added to the Prometheus stack.

This is extremely simple to be honest.

## Requirements

- Complete step [02-Add_Istio_Scrapping_Metrics](../02-Add_Istio_Scrapping_Metrics)

## Grafana

### Default credentials

> **Note:** \
> Since Grafana has no storage/volume, **all changes will be lost**

User: admin
Password: prom-operator

Just check any dashboard to see if it's working correctly.

I personally recommend the dashboard:

- **Node Exporter / USE Method / Node**

Lists the resource utilization for each one of the Nodes.

IDK check whatever you want, there are some good predefined graphs already.

### Want to change crededntials?

Just log into the admin user and change whatever the hell you want.

Username, email, password.

Select different preferences..., whatever.

### Want to manage/create Users/Teams?

Select `Administration` > `Users and Access`.

There you will be able to create/manage **Users**, **Teams** and **Service Accounts**.

### Istio related Dashboards

Here is a list of ready to go Istio related dashboards that you might want to set up on your Grafana Deployment.

- https://grafana.com/grafana/dashboards/7630-istio-workload-dashboard/
- https://grafana.com/grafana/dashboards/7636-istio-service-dashboard/
- https://grafana.com/grafana/dashboards/7645-istio-control-plane-dashboard/
- https://grafana.com/grafana/dashboards/7639-istio-mesh-dashboard/

The dashboards where found here:

- https://grafana.com/orgs/istio/dashboards


