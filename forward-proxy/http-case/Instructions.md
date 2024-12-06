# Tunnel the http traffic via a forward proxy.

This folder setup a forward proxy, that denies the traffic to http://www.google.com. Then ISTIO configurations are used to redirect the traffic via this proxy. 

### Instructions to setup:
1. Deploy the nginx forward proxy with deny rule.
```
docker compose -f setup/docker-compose.yml up -d
```
2. Deploys the istio configurations to redirect the traffic via proxy.
```
kubectl apply -f istio-proxy-config/proxy_config_http.yaml
```

### Verify:
Exec inside the pod and do a curl on http://www.google.com.

Expected is 404 error: 
```
 curl http://www.google.com/ -v
* Host www.google.com:80 was resolved.
* IPv6: 2404:6800:4015:800::2004
* IPv4: 142.250.70.132
*   Trying [2404:6800:4015:800::2004]:80...
* Immediate connect fail for 2404:6800:4015:800::2004: Network unreachable
*   Trying 142.250.70.132:80...
* Connected to www.google.com (142.250.70.132) port 80
* using HTTP/1.x
> GET / HTTP/1.1
> Host: www.google.com
> User-Agent: curl/8.11.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 404 Not Found
< server: envoy
< date: Fri, 06 Dec 2024 07:53:18 GMT
< content-type: text/html
< content-length: 162
< x-envoy-upstream-service-time: 11
<
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
```

### Cleanup:
```
docker compose -f setup/docker-compose.yml down
kubectl delete -f istio-proxy-config/proxy_config_http.yaml
```