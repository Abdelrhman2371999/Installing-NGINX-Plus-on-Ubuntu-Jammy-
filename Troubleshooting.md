Kubernetes NGINX Gateway Fabric + MetalLB Troubleshooting Guide
Overview

This document describes the full setup, debugging process, and resolution of issues encountered while deploying NGINX Gateway Fabric with Gateway API, MetalLB, and application routing in Kubernetes.

It covers:

Namespace and resource placement

Service & Endpoint issues

MetalLB networking problems

Gateway API (Gateway / HTTPRoute) behavior

Host-based routing pitfalls

Final working configuration

Cluster Environment

Kubernetes version: v1.30.x

Nodes:

k8s-master (10.10.0.200)

k8s-worker01 (10.10.0.207)

k8s-worker02 (10.10.0.204)

CNI: Pod CIDRs 192.168.x.x

LoadBalancer implementation: MetalLB (Layer2)

Gateway Controller: NGINX Gateway Fabric v2.3.0 (NGINX Plus)

Namespaces Used
Namespace	Purpose
nginx-gateway	Gateway controller, Gateway, HTTPRoutes, app
metallb-system	MetalLB
kube-system	Core components
default	Legacy/testing resources

⚠️ Important Rule
Kubernetes resources must live in the same namespace to work together:

Deployment

Service

ConfigMap

HTTPRoute

Gateway

Initial Problems Encountered
1. Ingress / Gateway Not Working

Services had no endpoints

Pods stuck in ContainerCreating

Requests returned:

503

404

connection refused

2. Service ClusterIP Conflicts

Errors such as:

failed to allocate IP: provided IP is already allocated


Cause:
Hardcoded clusterIP values in Service YAML.

Fix:
Remove clusterIP field and let Kubernetes assign it automatically.

MetalLB Issues
Symptoms

LoadBalancer IP reachable by ARP but refused connections

ip neigh showed FAILED

NodePort not responding

Diagnosis

Service selector did not match any Pods

Endpoints were empty

TargetPort mismatch

Fixes Applied

Correct Service selectors

kubectl patch svc gateway-service -n nginx-gateway \
--type=merge -p '
spec:
  selector:
    app.kubernetes.io/instance: ngf
    app.kubernetes.io/name: test-gateway-nginx
'


Fix targetPort
Pods were listening on port 80, but Service used 8080.

kubectl patch svc gateway-service -n nginx-gateway --type=json -p='[
  {"op":"replace","path":"/spec/ports/0/targetPort","value":80},
  {"op":"replace","path":"/spec/ports/1/targetPort","value":80}
]'


Verify Endpoints

kubectl get endpoints gateway-service -n nginx-gateway


Result:

192.168.xx.xx:80

Gateway API Configuration
Gateway
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

HTTPRoute (Final Correct Version)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: nginx-gateway
spec:
  parentRefs:
  - name: test-gateway
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: test-app
      port: 80

Hostname Routing Pitfall (IMPORTANT)
What Happened

Requests with:

Host: anything.local


returned NGINX default page

Requests with:

Host: 10.10.0.232


worked correctly

Why

When hostnames are defined in HTTPRoute, NGINX Gateway Fabric creates multiple server blocks:

One for the hostname

One default server

If the Host header does not match exactly, traffic goes to default server.

Fix Options
Option A — No hostname restriction (recommended for IP access)

Remove hostnames entirely:

kubectl patch httproute app-route --type=json -p='[
  {"op":"remove","path":"/spec/hostnames"}
]'

Option B — Enforce hostname
spec:
  hostnames:
  - app.local


And test with:

curl http://10.10.0.232 -H "Host: app.local"

Verifying NGINX Internals
Check listening ports
kubectl exec -n nginx-gateway deploy/test-gateway-nginx -- \
netstat -lnt


Expected:

0.0.0.0:80
0.0.0.0:8081

Inspect generated NGINX config
kubectl exec -n nginx-gateway deploy/test-gateway-nginx -- \
nginx -T


Look for:

server_name

listen 80

proxy_pass http://nginx-gateway_test-app_80

Final Working State
External Access
curl http://10.10.0.232/

NodePort Access
curl http://10.10.0.204:30523/

Internal Cluster Access
kubectl run tmp --rm -it --image=curlimages/curl -- \
curl http://gateway-service.nginx-gateway.svc.cluster.local

Result

✅ Requests are routed to test-app
✅ Gateway API working
✅ MetalLB LoadBalancer working
✅ No 503 / connection refused

Key Lessons Learned

Service selector must match Pod labels

targetPort must match container port

MetalLB only advertises IP — kube-proxy still matters

Gateway API is Host-header sensitive

Remove hostnames if accessing by IP

Always check Endpoints before debugging networking

Recommended Best Practice Layout
nginx-gateway/
├── gateway.yaml
├── httproute.yaml
├── app-deployment.yaml
├── app-service.yaml


Keep Gateway + HTTPRoute + Service + Deployment in same namespace

Use hostnames only when you control DNS

Status

🟢 Resolved & Stable

If you want:

TLS

HTTPS listener

Multiple apps

Path-based routing

Production hardening

Just ask.
