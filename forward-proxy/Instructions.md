# Tunnel the http traffic via a forward proxy.

This folder setup a forward proxy, that denies the traffic to http://www.google.com. Then ISTIO configurations are used to redirect the traffic via this proxy. 

### Instructions to setup:
1. Deploy the nginx forward proxy with deny rule.
```
docker compose -f setup/docker-compose.yml up -d
```
2. Deploys the istio configurations to redirect the http traffic via proxy.
```
kubectl apply -f http-case/proxy_config_http.yaml
```

### Verify HTTP Traffic:
Exec inside the pod and do a curl on http://www.google.com.

Expected is 403 error: 
```
curl http://www.google.com/ -v
* Host www.google.com:80 was resolved.
* IPv6: 2404:6800:400a:80b::2004
* IPv4: 142.250.70.164
*   Trying [2404:6800:400a:80b::2004]:80...
* Immediate connect fail for 2404:6800:400a:80b::2004: Network unreachable
*   Trying 142.250.70.164:80...
* Connected to www.google.com (142.250.70.164) port 80
* using HTTP/1.x
> GET / HTTP/1.1
> Host: www.google.com
> User-Agent: curl/8.11.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 403 Forbidden
< server: envoy
< date: Fri, 06 Dec 2024 21:14:04 GMT
< content-type: text/html
< content-length: 162
< x-envoy-upstream-service-time: 6
<
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
* Connection #0 to host www.google.com left intact
```

### Cleanup:
```
docker compose -f setup/docker-compose.yml down
kubectl delete -f istio-proxy-config/proxy_config_http.yaml
```