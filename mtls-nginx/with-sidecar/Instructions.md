# Deploy a mutual TLS server

Same as [with-gw](/mtls-nginx/with-gw/Instructions.md)

## Deploy side car resources command line 
```
kubectl create secret generic client-credential --from-file=tls.key=client.example.com.key --from-file=tls.crt=client.example.com.crt --from-file=ca.crt=example.com.crt

kubectl create role client-credential-role --resource=secret --verb=list
kubectl create rolebinding client-credential-role-binding --role=client-credential-role --serviceaccount=default:curl

kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: originate-mtls-for-nginx
spec:
  hosts:
  - my-nginx.mesh-external.svc.cluster.local
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
    targetPort: 443
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: originate-mtls-for-nginx
spec:
  workloadSelector:
    matchLabels:
      app: curl
  host: my-nginx.mesh-external.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    portLevelSettings:
    - port:
        number: 80
      tls:
        mode: MUTUAL
        credentialName: client-credential # this must match the secret created earlier to hold client certs, and works only when DR has a workloadSelector
        sni: my-nginx.mesh-external.svc.cluster.local
        # subjectAltNames: # can be enabled if the certificate was generated with SAN as specified in previous section
        # - my-nginx.mesh-external.svc.cluster.local
EOF
```
## Install from yamls:
You can simply do 
```
kubectl apply -f mtls-nginx/with-sidecar/.
```
## Verify
```
istioctl proxy-config secret deploy/curl | grep client-credential

curl http://my-nginx.mesh-external.svc.cluster.local -v
```
## Cleanup:
```
kubectl delete secret nginx-server-certs nginx-ca-certs -n mesh-external
kubectl delete secret client-credential
kubectl delete rolebinding client-credential-role-binding
kubectl delete role client-credential-role
kubectl delete configmap nginx-configmap -n mesh-external
kubectl delete service my-nginx -n mesh-external
kubectl delete deployment my-nginx -n mesh-external
kubectl delete namespace mesh-external
kubectl delete serviceentry originate-mtls-for-nginx
kubectl delete destinationrule originate-mtls-for-nginx
```
or
```
kubectl delete -f mtls-nginx/with-sidecar/.
```