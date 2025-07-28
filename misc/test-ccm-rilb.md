Quick test to see if I can create and reference a self-signed cert persisted in CCM.

```
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
   name: gateway
   namespace: gw-shared
spec:
   gatewayClassName: gke-l7-rilb
   listeners:
    - name: gateway-pre-shared-cert
      protocol: HTTPS
      port: 443
      tls:
         mode: Terminate
         options:
           networking.gke.io/cert-manager-certs: store-chilm-com-cert
      allowedRoutes:
        kinds:
         - kind: HTTPRoute
        namespaces:
           from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: default-httproute
  namespace: default
spec:
  parentRefs:
  - name: gateway
    namespace: gw-shared
    sectionName: gateway-pre-shared-cert
  hostnames:
  - "store.chilm.com"
  rules:
  - matches:
    - path:
        value: /
    backendRefs:
    - name: store-v1
      port: 8080
```
