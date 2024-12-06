kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: proxy
spec:
  hosts:
  - host.docker.internal
  exportTo:
  - "."
  addresses: 
  - 192.168.65.254/32
  ports:
  - number: 3128
    name: tcp
    protocol: TCP
  location: MESH_EXTERNAL
  resolution: NONE
EOF


kubectl create secret generic pingid-credential --from-file=tls.key=decrypted_key.pem --from-file=tls.crt=client_cert.pem --from-file=ca.crt=ca.pem

kubectl create role pingid-credential-role --resource=secret --verb=list
kubectl create rolebinding pingid-credential-role-binding --role=pingid-credential-role --serviceaccount=default:curl



kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: originate-mtls-for-pingid
spec:
  hosts:
  - external.service.com
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
  name: originate-mtls-for-pingid
spec:
  workloadSelector:
    matchLabels:
      app: curl
  host: external.service.com
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    portLevelSettings:
    - port:
        number: 80
      tls:
        mode: MUTUAL
        credentialName: pingid-credential # this must match the secret created earlier to hold client certs, and works only when DR has a workloadSelector
        sni: external.service.com
EOF



kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: proxy
spec:
  hosts:
  -  # ignored
  addresses:
  - corporate-proxy.com/32
  exportTo: ##The ’exportTo’ field allows for control over the visibility of a service declaration to other namespaces in the mesh. By default, a service is exported to all namespaces. The following example restricts the visibility to the current namespace, represented by “.”, so that it cannot be used by other namespaces.
  - "."
  ports:
  - number: 8080
    name: tcp
    protocol: TCP
  location: MESH_EXTERNAL
EOF


kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: mtls-to-external
spec:
  host: external.service.com
  exportTo: ##The ’exportTo’ field allows for control over the visibility of a service declaration to other namespaces in the mesh. By default, a service is exported to all namespaces. The following example restricts the visibility to the current namespace, represented by “.”, so that it cannot be used by other namespaces.
  - "."
  workloadSelector:
    matchLabels:
      app: curl
  trafficPolicy:
    tls:
      credentialName: pingid-credential #The name of the secret that holds the TLS certs for the client including the CA certificates. 
      mode: MUTUAL
EOF


