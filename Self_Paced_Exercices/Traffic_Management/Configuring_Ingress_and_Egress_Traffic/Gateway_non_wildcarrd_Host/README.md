
Create a Gateway that accepts traffic only when the **Host** Header 'patrick.star' is used.

Make virtual service that doesn't filter by host, and returns 200 to all the requests. Body "<h1>Sponge Bob</h1>"

```shell
➜  abcd curl 172.18.10.10 -Hhost:patrick.star 
<h1>SpongeBob_reply</h1>%

➜  abcd curl 172.18.10.10 -I -Hhost:patrick.starr
HTTP/1.1 404 Not Found
date: Sat, 19 Jul 2025 21:48:55 GMT
server: istio-envoy
transfer-encoding: chunked

➜  abcd curl 172.18.10.10 -I                     
HTTP/1.1 404 Not Found
date: Sat, 19 Jul 2025 21:48:57 GMT
server: istio-envoy
transfer-encoding: chunked
```