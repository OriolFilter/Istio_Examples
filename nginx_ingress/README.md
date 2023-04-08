
##### https://github.com/istio/istio/tree/master/samples

```shell
$ kubectl get ingress 
NAME             CLASS    HOSTS              ADDRESS        PORTS   AGE
demo-localhost   nginx    demo.localdev.me   192.168.1.31   80      21h
$ curl 192.168.1.31 
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
$ curl 192.168.1.31 -HHOST:demo.localdev.me 
<html><body><h1>It works!</h1></body></html>
```


https://kubernetes.github.io/ingress-nginx/user-guide/basic-usage/

ingress-nginx

https://docs.nginx.com/nginx-ingress-controller/