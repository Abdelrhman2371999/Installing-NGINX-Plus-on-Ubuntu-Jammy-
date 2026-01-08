 🚀 Quick Start (Minimal Setup)

This section is for users who just want the **NGINX Gateway Fabric + Gateway API** working quickly, without reading the troubleshooting details.

---

## Prerequisites

- Kubernetes cluster is running
- MetalLB installed and working (Layer2)
- NGINX Gateway Fabric installed
- `kubectl` access to the cluster

---

## 1️⃣ Verify MetalLB IP Pools

```bash
kubectl -n metallb-system get ipaddresspool
````

Make sure an IP range (example `10.10.0.231-10.10.0.240`) is available.

---

## 2️⃣ Verify NGINX Gateway Fabric Control Plane

```bash
kubectl -n nginx-gateway get pods
kubectl -n nginx-gateway get svc ngf-nginx-gateway-fabric -o wide
kubectl -n nginx-gateway get endpoints ngf-nginx-gateway-fabric -o wide
```

Expected:

* Pod `ngf-nginx-gateway-fabric` → `Running`
* Endpoint points to `:8443`

If needed (one-time fix):

```bash
kubectl -n nginx-gateway patch svc ngf-nginx-gateway-fabric --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/ports",
    "value": [
      {"name":"grpc","port":443,"targetPort":8443,"protocol":"TCP"}
    ]
  }
]'
```

---

## 3️⃣ Create Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: test-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
```

```bash
kubectl apply -f gateway.yaml
```

Verify:

```bash
kubectl -n nginx-gateway describe gateway test-gateway
```

---

## 4️⃣ Create HTTPRoute (No Hostnames – Accept All)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: nginx-gateway
spec:
  parentRefs:
  - name: test-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: test-app
      port: 80
```

```bash
kubectl apply -f httproute.yaml
```

---

## 5️⃣ Fix gateway-service (Critical)

Ensure the Service targets **port 80** (data-plane listens on 80):

```bash
kubectl -n nginx-gateway patch svc gateway-service --type='json' -p='[
  {"op":"replace","path":"/spec/ports/0/targetPort","value":80},
  {"op":"replace","path":"/spec/ports/1/targetPort","value":80}
]'
```

Verify endpoints:

```bash
kubectl -n nginx-gateway get endpoints gateway-service -o wide
```

Expected:

```
<PodIP>:80
```

---

## 6️⃣ Test Connectivity

### Inside cluster

```bash
kubectl -n nginx-gateway run tmp --rm -i --restart=Never --image=curlimages/curl -- \
  curl http://gateway-service.nginx-gateway.svc.cluster.local/
```

### From node / external

```bash
curl http://<MetalLB-IP>/
```

Example:

```bash
curl http://10.10.0.232/
```

Expected:

```
HTTP/1.1 200 OK
```

---

## ✅ Success Criteria

* Gateway status: **Accepted / Programmed**
* HTTPRoute status: **Accepted / ResolvedRefs**
* `curl http://<LB-IP>/` returns app response
* No empty endpoints
* No connection refused on NodePort or LoadBalancer IP

---

👉 If anything fails, continue reading the **Troubleshooting & Root Cause** sections below.

```
