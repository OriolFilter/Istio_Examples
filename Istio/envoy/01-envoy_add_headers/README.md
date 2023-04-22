https://github.com/istio/istio/wiki/EnvoyFilter-Samples

https://stackoverflow.com/questions/73262158/how-to-apply-envoyfilter-to-sidecar-inbound-and-gateway


https://istio.io/latest/docs/reference/config/networking/envoy-filter/

https://discuss.istio.io/t/adding-custom-response-headers-using-istios-1-6-0-envoy-lua-filter/7494



https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter


> kubectl logs -f deployments/istiod -n istio-system   



This somewhat is monitoring, can do cool stuff I don't know how or what to do


enable export access logs to stdout


istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout



https://istio.io/latest/docs/ops/diagnostic-tools/component-logging/




https://dev.to/aws-builders/understanding-istio-access-logs-2k5o

```yaml
Note: Here I am using request_handle:logCritical method because default logLevel is WARN for Istio components. request_handle:logInfo can be used, if logLevel is set to Info.
```

https://youtu.be/yOtEG1luTwU


