## Configure mutual TLS origination for egress traffic to external api (command line)
```
kubectl create secret -n istio-system generic mtlsapi-credential --from-file=tls.key=my_client.key --from-file=tls.crt=my_client.pem --from-file=ca.crt=server_cert.pem


kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mtlsapi
spec:
  hosts:
  - "mtlsapi.aspnet4you.com"
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - mtlsapi.aspnet4you.com
    tls:
      mode: ISTIO_MUTUAL
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: egressgateway-for-mtlsapi
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: mtlsapi
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
      portLevelSettings:
      - port:
          number: 443
        tls:
          mode: ISTIO_MUTUAL
          sni: mtlsapi.aspnet4you.com
EOF


kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: direct-mtlsapi-through-egress-gateway
spec:
  hosts:
  - mtlsapi.aspnet4you.com
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: mtlsapi
        port:
          number: 443
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 443
    route:
    - destination:
        host: mtlsapi.aspnet4you.com
        port:
          number: 443
      weight: 100
EOF

kubectl apply -n istio-system -f - <<EOF
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: originate-mtls-for-mtlsapi
spec:
  host: mtlsapi.aspnet4you.com
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    portLevelSettings:
    - port:
        number: 443
      tls:
        mode: MUTUAL
        credentialName: mtlsapi-credential # this must match the secret created earlier to hold client certs
        sni: mtlsapi.aspnet4you.com
        # subjectAltNames: # can be enabled if the certificate was generated with SAN as specified in previous section
        # - mtlsapi.aspnet4you.com
EOF
```
### Setup via yamls:
```
 kubectl apply -f mtls-external/with-gw/.
```
## Verify: 
istioctl -n istio-system proxy-config secret deploy/istio-egressgateway | grep mtlsapi-credential

curl http://mtlsapi.aspnet4you.com/pets -v

## Cleanup:
```
kubectl delete secret mtlsapi-credential -n istio-system
kubectl delete gw istio-egressgateway
kubectl delete virtualservice direct-mtlsapi-through-egress-gateway
kubectl delete destinationrule -n istio-system originate-mtls-for-mtlsapi
kubectl delete destinationrule egressgateway-for-mtlsapi
```
or

```
 kubectl delete -f mtls-external/with-gw/.
```



