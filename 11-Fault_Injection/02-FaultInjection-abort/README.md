---
gitea: none
include_toc: true
---

# Description

Based on the [previous example](../02-FaultInjection-abort), where we configure a "fault" that will make the backend take 10 more seconds before receiving the request, this time will make the request abort, resulting in a failed request (503 status code).

This will be applied to a 90% of the incoming traffic that matches the rule and will allow to confirm in a secure environment how the application would behave in such difficult situations, and apply the modifications required to avoid issue in case there would be a network issue.


This example configures:

    Generic Kubernetes resources:
    - 1 Service
    - 1 Deployments
    
    Istio resources:
    - 1 Gateway
    - 1 Virtual Service


# Based on

- [02-FaultInjection-abort](../02-FaultInjection-abort)
- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPFaultInjection-Abort

# Configuration

## Service

Creates a service named `helloworld`.

This service listens for the port `80` expecting `HTTP` traffic and will forward the incoming traffic towards the port `80` from the destination pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
    service: helloworld
spec:
  ports:
    - port: 80
      name: http
  selector:
    app: helloworld
```

## Deployment

Deploys a Nginx server that listens for the port `80`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-nginx
  labels:
    app: helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          image: nginx
          resources:
            requests:
              cpu: "100m"
          imagePullPolicy: IfNotPresent #Always
          ports:
            - containerPort: 80
```

## Gateway

Deploys an Istio gateway that's listening to the port `80` for `HTTP` traffic.

It doesn't filter for any specific host.

The `selector` field is used to "choose" which Istio Load Balancers will have this gateway assigned to.

The Istio `default` profile creates a Load Balancer in the namespace `istio-system` that has the label `istio: ingressgateway` set, allowing us to target that specific Load Balancer and assign this gateway resource to it.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: helloworld-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

## VirtualService

The Virtual Service resources are used to route and filter the received traffic from the gateway resources, and route it towards the desired destination.

On this example we select the gateway `helloworld-gateway`, which is the [gateway that 's described in the `Gateway` section](#gateway).

On this resource, we are also not limiting the incoming traffic to any specific host, allowing for all the incoming traffic to go through the rules set.

Here we created a rule that will be applied on `HTTP` related traffic (including `HTTPS` and `HTTP2`) when the destination path is exactly `/helloworld`.

This traffic will be forwarded to the port `80` of the destination service `helloworld` (the full path URL equivalent would be `helloworld.$NAMESPACE.svc.cluster.local`).

There will be an internal URL rewrite set, as if the URL is not modified, it would attempt to reach to the `/helloworld` path from the Nginx deployment, which currently has no content and would result in an error code `404` (Not found).

Additionally, we apply a "fault", where a 90% of the traffic will be aborted and receive a 503 status code.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-vs
spec:
  hosts:
    - "*"
  gateways:
    - helloworld-gateway
  http:
    - match:
        - uri:
            exact: /helloworld
      route:
        - destination:
            host: helloworld
            port:
              number: 80
      rewrite:
        uri: "/"
      fault:
        abort:
          percentage:
            value: 90
          httpStatus: 503
```

# Walkthrough

## Deploy resources

Deploy the resources.

```shell
kubectl apply -f ./
```
```text 
deployment.apps/helloworld-nginx created
gateway.networking.istio.io/helloworld-gateway created
service/helloworld created
virtualservice.networking.istio.io/helloworld-vs created
```

## Wait for the pods to be ready

Wait for the Nginx deployments to be up and ready.

```shell
kubectl get deployment helloworld-nginx -w
```
```text 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-nginx   1/1     1            1           12s
```

## Test the service

### Get LB IP

To perform the desired tests, we will need to obtain the IP Istio Load Balancer that we selected in the [Gateway section](#gateway).

On my environment, the IP is the `192.168.1.50`.

```shell
kubectl get svc -l istio=ingressgateway -A
```
```text
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.97.47.216   192.168.1.50   15021:31316/TCP,80:32012/TCP,443:32486/TCP   39h
```

### helloworld

We will use the `curl` command and feed it a template to provide us with the status code from the request.

Since the fault that we set had a 90% chance of triggering, if you are "unlucky", and get instantly the response from the backend, you might need to run the command multiple times in order to get the fault triggered.

```shell
curl -w @- -o /dev/null -s 192.168.1.21/helloworld <<'EOF'
          http_code:  %{http_code}\n
                    ----------\n
         time_total:  %{time_total}\n
EOF
```

```text
          http_code:  503
                    ----------
         time_total:  0.037870
```

From the command output, we can observe that the request resulted in a 503 status code.

We could continue sending curls until we receive a successful `200` status code. 

## Cleanup

Finally, a cleanup from the resources deployed.

```shell
kubectl delete -f ./
```
```text
deployment.apps "helloworld-nginx" deleted
gateway.networking.istio.io "helloworld-gateway" deleted
service "helloworld" deleted
virtualservice.networking.istio.io "helloworld-vs" deleted
```

# Links of interest

- https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPFaultInjection-Abort