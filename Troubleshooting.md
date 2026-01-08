
# NGINX Gateway Fabric (Gateway API) + MetalLB — Full Troubleshooting & Fix Log

This README documents the full troubleshooting journey and final fixes for:
- NGINX Gateway Fabric (NGF) control-plane + data-plane
- Gateway API resources (Gateway + HTTPRoute)
- MetalLB LoadBalancer IP advertisement (Layer2)
- kube-proxy / nftables NodePort programming
- Host-based routing behaviors (why some Host headers showed the wrong page)

---

## Table of Contents
- [Environment](#environment)
- [Resources Overview](#resources-overview)
- [Initial Symptoms](#initial-symptoms)
- [Root Causes Found](#root-causes-found)
- [Fixes Applied (Chronological)](#fixes-applied-chronological)
  - [A) Service selector & empty endpoints](#a-service-selector--empty-endpoints)
  - [B) Data-plane not listening on expected ports](#b-data-plane-not-listening-on-expected-ports)
  - [C) NGF control-plane Service targetPort mismatch (grpc)](#c-ngf-control-plane-service-targetport-mismatch-grpc)
  - [D) gateway-service targetPort mismatch](#d-gateway-service-targetport-mismatch)
  - [E) Hostname routing confusion](#e-hostname-routing-confusion)
- [Verification Commands](#verification-commands)
- [Final Working Config (Recommended)](#final-working-config-recommended)
- [Common Pitfalls & Notes](#common-pitfalls--notes)

---

## Environment

- Nodes:
  - `k8s-master` (10.10.0.200)
  - `k8s-worker01` (10.10.0.207)
  - `k8s-worker02` (10.10.0.204)
- Pod CIDR range: `192.168.x.x`
- LoadBalancer: **MetalLB (layer2)**
- kube-proxy: DaemonSet running on all nodes
- NGINX Gateway Fabric:
  - control plane: `ngf-nginx-gateway-fabric`
  - data plane: `test-gateway-nginx` (NGINX Plus)

---

## Resources Overview

### Services
- `nginx-gateway/gateway-service` (LoadBalancer)
  - External IP (MetalLB): `10.10.0.232`
  - NodePort: `30523` (for port 80)
- `nginx-gateway/ngf-nginx-gateway-fabric` (LoadBalancer)
  - ClusterIP: `10.100.66.136`
  - External IP (MetalLB): `10.10.0.231`
  - Used by NGINX Agent / control-plane gRPC

### Gateway API
- `nginx-gateway/Gateway test-gateway`
  - Listener: HTTP :80
- `nginx-gateway/HTTPRoute app-route`
  - Backend: `test-app:80`

---

## Initial Symptoms

### 1) `gateway-service` had no endpoints
```bash
kubectl -n nginx-gateway get endpoints gateway-service -o wide
# ENDPOINTS was empty
````

### 2) Data-plane pod was Running but NOT Ready

Readiness probe failed:

* `http://<pod-ip>:8081/readyz` connection refused

### 3) External connectivity failed (NodePort / LoadBalancer)

Examples:

* `curl http://10.10.0.232:80` → connection refused
* `curl http://10.10.0.204:30523` → connection refused

### 4) NGINX Agent could not connect to control plane

Logs showed repeated timeouts to:

* `dial tcp 10.100.66.136:443: i/o timeout`

### 5) Host header changed the response unexpectedly

* `Host: 10.10.0.232` returned the app page ✅
* `Host: anything.local` returned the default nginx welcome page ❌

---

## Root Causes Found

1. **Empty endpoints** because Service selector/targetPort didn’t match the real data-plane ports.
2. **Data-plane was not listening on the Service targetPort** (Service sent traffic to 8080 but container listened elsewhere).
3. **Control-plane service (`ngf-nginx-gateway-fabric`) was pointing to the wrong targetPort**, breaking NG Agent ↔ control-plane connectivity.
4. **Hostname routing rules created different NGINX server blocks**, so different Host headers hit different server blocks.

---

## Fixes Applied (Chronological)

### A) Service selector & empty endpoints

Check service and endpoints:

```bash
kubectl -n nginx-gateway describe svc gateway-service
kubectl -n nginx-gateway get endpoints gateway-service -o wide
```

If endpoints are empty, verify the selector matches the pod labels:

```bash
kubectl -n nginx-gateway get pods --show-labels | grep test-gateway-nginx
kubectl -n nginx-gateway get svc gateway-service -o yaml
```

> ✅ After aligning selectors/labels and recreating pods, endpoints began to appear.

---

### B) Data-plane not listening on expected ports

We inspected listening ports inside the data-plane container:

```bash
kubectl -n nginx-gateway exec -it deploy/test-gateway-nginx -- sh -lc '
(netstat -lnt 2>/dev/null || (apk add --no-cache net-tools >/dev/null 2>&1 && netstat -lnt)) \
| egrep ":(80|8080|8081)\b"'
```

Result showed the container listens on:

* `:80`
* `:8081` (readyz)

So the Service **must target port 80** (not 8080).

---

### C) NGF control-plane Service targetPort mismatch (grpc)

The NG Agent logs showed it tried to connect to:

* `10.100.66.136:443` (service clusterIP)

But the Service was not targeting the correct control-plane port.

We confirmed the endpoint target for `ngf-nginx-gateway-fabric`:

```bash
kubectl -n nginx-gateway get endpoints ngf-nginx-gateway-fabric -o wide
```

Fix applied: patch the control-plane Service to map:

* `port 443 -> targetPort 8443`

```bash
kubectl -n nginx-gateway patch svc ngf-nginx-gateway-fabric --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/ports",
    "value": [
      {
        "name": "grpc",
        "port": 443,
        "targetPort": 8443,
        "protocol": "TCP"
      }
    ]
  }
]'
```

Verify:

```bash
kubectl -n nginx-gateway get svc ngf-nginx-gateway-fabric -o wide
kubectl -n nginx-gateway get endpoints ngf-nginx-gateway-fabric -o wide
# should show something like: <pod-ip>:8443
```

---

### D) gateway-service targetPort mismatch

After confirming the data-plane listens on **:80**, we patched `gateway-service` targetPorts to **80**.

```bash
kubectl -n nginx-gateway patch svc gateway-service --type='json' -p='[
  {"op":"replace","path":"/spec/ports/0/targetPort","value":80},
  {"op":"replace","path":"/spec/ports/1/targetPort","value":80}
]'
```

Verify endpoints now show `:80`:

```bash
kubectl -n nginx-gateway get ep gateway-service -o wide
```

Now connectivity succeeded:

```bash
curl -v http://10.10.0.232:80/
curl -v http://10.10.0.204:30523/
curl -v http://10.99.191.190:80/
```

---

### E) Hostname routing confusion

#### What we observed

* `curl http://10.10.0.232/ -H 'Host: 10.10.0.232'` returned app ✅
* `curl http://10.10.0.232/ -H 'Host: anything.local'` returned nginx welcome ❌

#### Why

When you set `spec.hostnames` in HTTPRoute, NGF generates a server block for that hostname.
Requests that don’t match that hostname may fall into a different server (default or regex host),
which can serve a different response.

#### The “correct” choice depends on what you want:

##### Option 1 (Recommended): accept ANY host header

Do **not** set `spec.hostnames` in HTTPRoute.

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

##### Option 2: enforce a hostname (requires DNS / hosts file)

If you set:

```yaml
spec:
  hostnames:
  - app.local
```

Then your test MUST include:

```bash
curl http://10.10.0.232/ -H 'Host: app.local'
```

> If you want “all URLs work”, you need Option 1 (no hostnames),
> OR you need DNS to always send the same hostname.

---

## Verification Commands

### 1) Confirm services & endpoints

```bash
kubectl -n nginx-gateway get svc -o wide
kubectl -n nginx-gateway get endpoints -o wide
```

### 2) Confirm data-plane listens on 80

```bash
kubectl -n nginx-gateway exec -it deploy/test-gateway-nginx -- sh -lc '
netstat -lnt 2>/dev/null || (apk add --no-cache net-tools >/dev/null 2>&1 && netstat -lnt)'
```

### 3) Inspect generated NGINX config (server blocks, proxy_pass)

```bash
kubectl -n nginx-gateway exec -it deploy/test-gateway-nginx -- sh -lc \
"nginx -T 2>/dev/null | egrep -n 'listen 80|server_name|proxy_pass|upstream' | head -200"
```

### 4) Confirm Gateway/HTTPRoute status

```bash
kubectl -n nginx-gateway describe gateway test-gateway
kubectl -n nginx-gateway describe httproute app-route
```

### 5) kube-proxy / nftables sanity checks (NodePort)

```bash
kubectl -n kube-system get ds kube-proxy -o wide
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
sudo nft list ruleset | egrep "dport 30523|KUBE-EXTERNAL-SERVICES" -n
```

---

## Final Working Config (Recommended)

### Gateway

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

### HTTPRoute (no hostnames)

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

### Key Service Fixes

#### gateway-service targets 80

```bash
kubectl -n nginx-gateway patch svc gateway-service --type='json' -p='[
  {"op":"replace","path":"/spec/ports/0/targetPort","value":80},
  {"op":"replace","path":"/spec/ports/1/targetPort","value":80}
]'
```

#### control-plane service maps 443 -> 8443

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

## Common Pitfalls & Notes

* `kubectl describe` does **NOT** support `-o wide` (that’s only for `kubectl get`).
* If `endpoints` is empty:

  * selector mismatch OR pods not ready OR wrong port name/targetPort
* If NGINX Agent can’t connect:

  * verify `ngf-nginx-gateway-fabric` Service targetPort matches real pod port
* If different Host headers return different pages:

  * you likely created multiple server blocks via `spec.hostnames`
* If `curl /foo` returns 404:

  * that may be **your app** returning 404 (not the gateway). Test your app directly via service.

---

## Result

✅ MetalLB LB IP works
✅ NodePort works
✅ Data-plane is Ready and serving traffic
✅ Gateway + HTTPRoute are Accepted / ResolvedRefs
✅ Routing works as expected when hostnames are configured correctly (or removed)

```
