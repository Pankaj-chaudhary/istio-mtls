# Connecting to External Services with mTLS and Istio

I am working on connecting to an external service from my app by offloading the mTLS responsibility to the service mesh. To test this locally, I followed the instructions in the Istio documentation and successfully configured mTLS origination from the sidecar for `kubernetes.docker.internal` and `my-nginx.mesh-external.svc.cluster.local`.

However, I now need to achieve the same with a service that can only be accessed through my corporate proxy (port 3128). I created the secret, ServiceEntry, and DestinationRule as mentioned in the documentation, but it is still not working.

- Application -> Sidecar -> mTLS nginx service on local: ✅
- Application -> Sidecar -> corporate-proxy (3128) -> mTLS external service: ❌

Note: From the pod, I can successfully perform `curl --cacert --cert --key https://externalapi` after setting the `HTTPS_PROXY` environment variable.

![Istio](istio.png)

- External API Reference: https://blogs.aspnet4you.com/2021/03/15/secure-your-business-api-with-mtls/
- External API is https://mtlsapi.aspnet4you.com/pets

# Pre-reqs to replicate the use cases in Repo:
1. Make sure you have a kubernetes cluster running (Managed from cloud, or even the one from docker).
2. Install istio using the demo [configuration profile](https://istio.io/latest/docs/setup/additional-setup/config-profiles/). 
3. To use the egress gateway for the tls/mtls orgination outside the mesh, make sure you have the egress gateway deployed.
4. Make sure you have the istio injection enabled for your namespace. 
```
kubectl label namespace default istio-injection=enabled
```
5. Deploy the curl pod to test all of this in the istio-injection enabled namespace, and check there are two containers running in the pod. [Curl Deployment](/test-deployment/curl.yaml)

# Deployment Stack
1. Non-Istio namespace, nginx proxy running over mtls. [Ref](/mtls-nginx/)
2. External API running over mtls [Ref](/mtls-external/)

# Current Status
### From personal computer:
- ✅ Application -> Sidecar -> mTLS nginx service on local
- ✅ Application -> Egress Gateway -> mTLS nginx service on local
- ✅ Application -> Sidecar -> mTLS public API
- ✅ Application -> Egress Gateway -> mTLS public API.
    - Seems related to [issue](https://discuss.istio.io/t/istio-mtls-to-an-external-service/12473).
    - Fixed by creating a [serviceentry](https://github.com/istio/istio/issues/30808#issuecomment-777675639).

### From corporate machine: 
- ✅ Application -> Sidecar -> mTLS nginx service on local
- ✅ Application -> Egress Gateway -> mTLS nginx service on local
- ⏳ Application -> Sidecar -> mTLS public API (To be verified, as i need the call to go via proxy)
- ⏳ Application -> Sidecar -> corporate proxy -> mTLS public API 
- ⏳ Application -> Egress Gateway -> corporate proxy -> mTLS public API 

### Note:
Got a lead to try Destination rule tunnel https://github.com/istio/istio/issues/41854