# Instructions to do mTLS via side car, and there will be no Egress gatway involved here.

### Setup manually command line:
```
 kubectl create secret generic client-credential --from-file=tls.key=my_client.key --from-file=tls.crt=my_client.pem --from-file=ca.crt=server_cert.pem

 kubectl create role client-credential-role --resource=secret --verb=list

 kubectl create rolebinding client-credential-role-binding --role=client-credential-role --serviceaccount=default:curl
```
### Setup via yamls:
```
 kubectl apply -f mtls-external/with-sidecar/.
```
### Verify
```
 istioctl proxy-config secret deploy/curl | grep client-credential
```
 Logs: 
```
 kubectl logs -l app=curl -c istio-proxy
```

 Curl: https://mtlsapi.aspnet4you.com/pets

 Outside the mesh: 
```
 curl --cacert server_cert.pem  --cert my_client.pem --key my_client.key https://mtlsapi.aspnet4you.com/pets -v
```
 From the mesh with DR and SE created:
```
 curl http://mtlsapi.aspnet4you.com/pets -v
```

## Cleanup:
```
kubectl delete secret client-credential
kubectl delete rolebinding client-credential-role-binding
kubectl delete role client-credential-role
```
or

```
 kubectl delete -f mtls-external/with-gw/.
```
