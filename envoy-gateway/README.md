# Envoy Gateway scenarios

- (m)TLS Passthrough
- mTLS validation and client cert forwarding

## Common set up

First we create a GKE Autopilot cluster and get the credentials...

Next we'll install Envoy Gateway into the GKE cluster:

```
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.4.2 -n envoy-gateway-system --create-namespace
```
(in this case the machinery will be installed into a namespace called `envoy-gateway-system`)

Later we'll deploy a `GatewayClass` called `eg`. When we deploy a `Gateway` with class `eg` by default it will be provisioned
as a `Service` of type `Loadbalancer` - without any additional annotations. Also only a single envoy proxy will be provisioned.
We can customize this by creating an [EnvoyProxy](https://gateway.envoyproxy.io/latest/tasks/operations/customize-envoyproxy/) yaml,
and associating it with either the `GatewayClass` or the `Gateway`.

First let's create a namespace for the Egress Gateway deployments (to separate platform concerns).

```
kubectl create ns shared-eg
```

Now let's apply the following:

```
kubectl apply -f - <<EOF
  apiVersion: gateway.envoyproxy.io/v1alpha1
  kind: EnvoyProxy
  metadata:
    name: custom-proxy-config
    namespace: shared-eg
  spec:
    provider:
      type: Kubernetes
      kubernetes:
        envoyService:
          annotations:
            networking.gke.io/load-balancer-type: "Internal"
            networking.gke.io/internal-load-balancer-allow-global-access: "true"
        envoyHpa:
          minReplicas: 3
          maxReplicas: 9
          metrics:
            - resource:
                name: cpu
                target:
                  averageUtilization: 60
                  type: Utilization
              type: Resource
EOF
```

## mTLS Passthru

Apply this to the `shared-eg` namespace

```
kubectl apply -n shared-eg -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: eg
  spec:
    gatewayClassName: eg
    infrastructure:
      parametersRef:
        group: gateway.envoyproxy.io
        kind: EnvoyProxy
        name: custom-proxy-config
    listeners:
      - name: tls
        protocol: TLS
        hostname: "passthrough.example.com"
        port: 6443
        tls:
          mode: Passthrough
        allowedRoutes:
          kinds:
          - kind: TLSRoute
EOF
```
Now we'll deploy a backend that will echo the request.

Create a namespace
```
kubectl create ns backend
```

First we'll create certs.

Note: These certificates will not be used by the Gateway, but will remain in the application scope.

Create a root certificate and private key to sign certificates:

```
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
```
Create a certificate and a private key for passthrough.example.com:

```
openssl req -out passthrough.example.com.csr -newkey rsa:2048 -nodes -keyout passthrough.example.com.key -subj "/CN=passthrough.example.com/O=some organization"
openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in passthrough.example.com.csr -out passthrough.example.com.crt
```

Store the cert/keys in a Secret:

```
kubectl create secret tls server-certs -n backend --key=passthrough.example.com.key --cert=passthrough.example.com.crt
```

Apply this to namespace backend
```
kubectl apply -n backend -f - <<EOF
  apiVersion: v1
  kind: Service
  metadata:
    name: passthrough-echoserver
    labels:
      run: passthrough-echoserver
  spec:
    ports:
      - port: 443
        targetPort: 8443
        protocol: TCP
    selector:
      run: passthrough-echoserver
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: passthrough-echoserver
  spec:
    selector:
      matchLabels:
        run: passthrough-echoserver
    replicas: 1
    template:
      metadata:
        labels:
          run: passthrough-echoserver
      spec:
        containers:
          - name: passthrough-echoserver
            image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            ports:
              - containerPort: 8443
            env:
              - name: HTTPS_PORT
                value: "8443"
              - name: TLS_SERVER_CERT
                value: /etc/server-certs/tls.crt
              - name: TLS_SERVER_PRIVKEY
                value: /etc/server-certs/tls.key
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            volumeMounts:
              - name: server-certs
                mountPath: /etc/server-certs
                readOnly: true
        volumes:
          - name: server-certs
            secret:
              secretName: server-certs
  ---
  apiVersion: gateway.networking.k8s.io/v1alpha2
  kind: TLSRoute
  metadata:
    name: tlsroute
  spec:
    parentRefs:
      - name: eg
        namespace: shared-eg
        sectionName: tls
    hostnames:
      - "passthrough.example.com"
    rules:
      - backendRefs:
          - group: ""
            kind: Service
            name: passthrough-echoserver
            port: 443
            weight: 1
EOF
```

Test by curling the IP address of the Gateway.

**Note** We provisioned an internal LB, so please have a bastion host in the same subnet to test. 

```
export GATEWAY_HOST=$(kubectl get gateway/eg -o jsonpath='{.status.addresses[0].value}')
```

Curl the example app through the Gateway, e.g. Envoy proxy:

`curl -v -HHost:passthrough.example.com --resolve "passthrough.example.com:6443:${GATEWAY_HOST}" \
--cacert example.com.crt https://passthrough.example.com:6443/get`
