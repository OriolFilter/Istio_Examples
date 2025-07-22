

Use a docker container with a https server.

Set up redirect from HTTP to HTTPS at the Gateway Level.



When testing and spoofing the header/SNI, following redirects **WON'T WORK**, since after doing the HTTP -> to HTTPS Istio will "resolve" the external host.


```shell
curl --insecure --resolve lb.net:80:172.18.10.10 http://lb.net -I  
```

```text
HTTP/1.1 301 Moved Permanently
location: https://lb.net/
date: Sat, 19 Jul 2025 23:52:51 GMT
server: istio-envoy
transfer-encoding: chunked
```

```shell
curl --insecure --resolve lb.net:443:172.18.10.10 https://lb.net  
```

```text
<h2>Howdy</h2>
```