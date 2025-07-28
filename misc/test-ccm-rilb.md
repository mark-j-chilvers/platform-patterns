Quick test to see if I can create and reference a self-signed cert persisted in CCM.

# Prereqs

- GKE cluster with GKE Gateway API enabled (Autopilot is by default)
- Created proxy subnet for the L7 ILB

# Deploy backend workload (in default NS)

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/gke-networking-recipes/main/gateway/gke-gateway-controller/app/store.yaml
```

# Create a cert in Certificate Manager

I created a self-signed cert and pushed it to CCM with:
```
gcloud certificate-manager certificates create "store-chilm-com-cert" \
    --certificate-file="CERTIFICATE_FILE" \
    --private-key-file="PRIVATE_KEY_FILE" \
    --location="REGION"
```
`REGION` matches where your cluster / ILB will be.

# Deploy the Gateway

Creating Gateway in its own NS (not necessary, more by convention as a centrally managed shared infra resource)
```
kubectl create ns gw-shared
```

Save the following as gateway-and-route.yaml
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
And now apply it

```
kubectl apply -f gateway-and-route.yaml
```

Test from a bastion in the same region:
```
curl https://store.chilm.com --resolve store.chilm.com:443:10.150.0.3 --cacert CERTIFICATE_FILE -v
```
(passing in the CA cert previously created)

