# NGINX Gateway Fabric (NGF) Installation and Testing Guide

This guide explains how to deploy NGINX Gateway Fabric (NGF) in a Kubernetes environment and test traffic routing using practical examples based on actual deployment scenarios.

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Installation Steps](#installation-steps)
4. [Verification and Testing](#verification-and-testing)
5. [Understanding Core Concepts](#understanding-core-concepts)
6. [Testing Methodology](#testing-methodology)
7. [Production Considerations](#production-considerations)
8. [Troubleshooting](#troubleshooting)

## Overview

**NGINX Gateway Fabric (NGF)** is a Kubernetes-native implementation of the Gateway API specification that uses NGINX Plus as the data plane. It provides advanced traffic management, load balancing, and security for applications running in Kubernetes clusters.

NGF operates by:
- Watching Kubernetes for Gateway API resources (Gateway, HTTPRoute)
- Translating these resources into NGINX Plus configuration
- Routing traffic from external clients to appropriate backend services

## Prerequisites

Before beginning installation, ensure you have:

1. **Kubernetes Cluster** (v1.24+ recommended)
2. **kubectl** configured with cluster access
3. **Helm** (v3.8.0+)
4. **NGINX Plus License** (trial or commercial)
5. **NGINX One Evaluation Token** (`nginx-one-eval.jwt` file)

## Installation Steps

### 1. Create Namespace and Secrets

```bash
# Create namespace for NGF
kubectl create namespace nginx-gateway

# Create secret for NGINX private registry
kubectl create secret docker-registry nginx-plus-registry-secret \
  --docker-server=private-registry.nginx.com \
  --docker-username=`cat nginx-one-eval.jwt` \
  --docker-password=none \
  -n nginx-gateway

# Create secret for NGINX Plus license
kubectl create secret generic nplus-license \
  --from-file license.jwt=nginx-one-eval.jwt \
  -n nginx-gateway
```

### 2. Check Available NGF Images

```bash
# List available NGINX Gateway Fabric docker images
curl -s https://private-registry.nginx.com/v2/nginx-gateway-fabric/nginx-plus/tags/list \
  --key nginx-one-eval.key \
  --cert nginx-one-eval.crt | jq
```

**Note:** Replace `nginx-one-eval.key` and `nginx-one-eval.crt` with your actual certificate and key file paths.

### 3. Apply Gateway API CRDs

```bash
# Apply Gateway API Custom Resource Definitions (CRDs)
# Replace v2.2.2 with the latest version from the previous step
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.2.2" | kubectl apply -f -
```

### 4. Install NGF via Helm

```bash
# Install NGINX Gateway Fabric using Helm
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --set nginx.image.repository=private-registry.nginx.com/nginx-gateway-fabric/nginx-plus \
  --set nginx.image.tag=2.2.2 \
  --set nginx.plus=true \
  --set serviceAccount.imagePullSecret=nginx-plus-registry-secret \
  --set nginx.imagePullSecret=nginx-plus-registry-secret \
  --set nginx.usage.secretName=nplus-license \
  --set nginx.service.type=NodePort \
  --set nginxGateway.snippetsFilters.enable=true \
  -n nginx-gateway
```

**Note:** Replace `2.2.2` with the latest version identified in step 2.

## Verification and Testing

### 1. Verify Installation

```bash
# Check NGF pod status
kubectl get pods -n nginx-gateway

# Expected output:
# NAME                                            READY   STATUS    RESTARTS   AGE
# ngf-nginx-gateway-fabric-bb7b4c469-85lpz        1/1     Running   0          6s

# Check NGF logs
kubectl logs -l app.kubernetes.io/instance=ngf -n nginx-gateway -c nginx-gateway

# Check service status
kubectl get svc -n nginx-gateway

# Check gatewayclass
kubectl get gatewayclass

# Expected output:
# NAME    CONTROLLER                                   ACCEPTED   AGE
# nginx   gateway.nginx.org/nginx-gateway-controller   True       3h45m
```

### 2. Create Sample Gateway and HTTPRoute Resources

Create a file named `sample-gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
```

Apply the gateway:

```bash
kubectl apply -f sample-gateway.yaml
```

### 3. Testing Commands Explained

The following commands demonstrate how to test NGF traffic routing:

```bash
# 1. Check Gateway API resources
kubectl get gateway -A
kubectl get httproute -A

# 2. Get NGF pod's node IP address
export NGF_IP=$(kubectl get pod -l app.kubernetes.io/instance=ngf -o json | jq '.items[0].status.hostIP' -r)

# 3. Get the HTTP NodePort for your gateway service
# Replace 'gateway-nginx' with your actual service name
export HTTP_PORT=$(kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}')

# Display the connection information
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"

# 4. Test HTTP traffic routing
# The --resolve flag bypasses DNS for testing
curl --resolve dashboard-sandbox.nafeza.gov.eg:$HTTP_PORT:$NGF_IP http://dashboard-sandbox.nafeza.gov.eg:$HTTP_PORT

# 5. For applications in specific namespaces
export NGF_IP=$(kubectl get pod -l app.kubernetes.io/instance=ngf -n your-app-namespace -o json | jq '.items[0].status.hostIP' -r)

# Example: Test coffee endpoint
curl --resolve dashboard-sandbox.nafeza.gov.eg:$HTTP_PORT:$NGF_IP http://dashboard-sandbox.nafeza.gov.eg:$HTTP_PORT/coffee

# 6. Test HTTPS traffic (if configured)
# Get HTTPS NodePort for your frontend service
export HTTPS_PORT=$(kubectl get svc nafeza-front-end -n nafeza-frontend -o jsonpath='{.spec.ports[1].nodePort}')

# Test HTTPS endpoint (using -k to skip certificate verification for testing)
curl -k --resolve dashboard-sandbox.nafeza.gov.eg:$HTTPS_PORT:$NGF_IP https://dashboard-sandbox.nafeza.gov.eg:$HTTPS_PORT

# Test HTTPS with specific path
curl -k --resolve dashboard-sandbox.nafeza.gov.eg:$HTTPS_PORT:$NGF_IP https://dashboard-sandbox.nafeza.gov.eg:$HTTPS_PORT/coffee
```

## Understanding Core Concepts

### Gateway API Resources

| Resource | Purpose | Example |
|----------|---------|---------|
| **GatewayClass** | Defines a class of Gateways with shared configuration | `nginx` gateway class |
| **Gateway** | Defines network endpoints (listeners) for traffic | Listens on port 80 for HTTP |
| **HTTPRoute** | Defines routing rules for HTTP traffic | Route `/api/*` to backend service |

### How Traffic Flows

```
External Client → NodeIP:NodePort → NGF Pod → Gateway → HTTPRoute → Kubernetes Service → Application Pods
```

### Key Testing Concepts

1. **NodePort Services**: NGF is exposed via NodePort, which maps a port on each cluster node to the NGF service.
2. **Host IP**: The IP address of the Kubernetes node where the NGF pod is running.
3. **curl --resolve**: A testing technique that bypasses DNS to send traffic directly to a specific IP and port.

## Testing Methodology

### Automated Testing Script

Create a script `test-ngf-routing.sh`:

```bash
#!/bin/bash

# Test NGF routing
echo "Testing NGINX Gateway Fabric Routing..."
echo "========================================"

# Get NGF information
NGF_IP=$(kubectl get pod -l app.kubernetes.io/instance=ngf -n nginx-gateway -o jsonpath='{.items[0].status.hostIP}')
HTTP_PORT=$(kubectl get svc ngf-nginx-gateway-fabric -n nginx-gateway -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
HTTPS_PORT=$(kubectl get svc ngf-nginx-gateway-fabric -n nginx-gateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

echo "NGF Node IP: $NGF_IP"
echo "HTTP NodePort: $HTTP_PORT"
echo "HTTPS NodePort: $HTTPS_PORT"
echo ""

# Test endpoints
TEST_HOST="dashboard-sandbox.nafeza.gov.eg"

echo "Testing HTTP endpoint..."
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" \
  --resolve $TEST_HOST:$HTTP_PORT:$NGF_IP \
  http://$TEST_HOST:$HTTP_PORT/health

echo ""
echo "Testing HTTPS endpoint..."
curl -k -s -o /dev/null -w "HTTPS Status: %{http_code}\n" \
  --resolve $TEST_HOST:$HTTPS_PORT:$NGF_IP \
  https://$TEST_HOST:$HTTPS_PORT/health

echo ""
echo "Testing complete."
```

### Sample Application Deployment

To test NGF routing, deploy a sample application:

```yaml
# sample-app.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: demo-app
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-service
  namespace: demo-app
spec:
  selector:
    app: demo-app
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: demo-app
spec:
  parentRefs:
  - name: my-gateway
    namespace: nginx-gateway
  hostnames:
  - "demo.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: demo-app-service
      port: 80
```

Apply and test:

```bash
kubectl apply -f sample-app.yaml

# Test the demo application
export DEMO_PORT=$(kubectl get svc ngf-nginx-gateway-fabric -n nginx-gateway -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
curl --resolve demo.example.com:$DEMO_PORT:$NGF_IP http://demo.example.com:$DEMO_PORT
```

## Production Considerations

### 1. Service Type Configuration

For production, consider using `LoadBalancer` instead of `NodePort`:

```bash
# Update service type during installation
helm upgrade ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --set nginx.service.type=LoadBalancer \
  --reuse-values \
  -n nginx-gateway
```

### 2. TLS/SSL Configuration

Configure proper TLS termination:

```yaml
# tls-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tls-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: tls-secret
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

### 3. Monitoring and Observability

Enable monitoring:

```bash
# Install Prometheus operator (if not already installed)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack

# Configure NGF metrics
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: ngf-metrics-config
  namespace: nginx-gateway
data:
  nginx-metrics.conf: |
    server {
      listen 9113;
      location /metrics {
        stub_status on;
      }
    }
EOF
```

## Troubleshooting

### Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **NGF pod not starting** | Pod status `CrashLoopBackOff` | Check logs: `kubectl logs -n nginx-gateway <pod-name>` |
| **Gateway not accepted** | `kubectl get gateway` shows `False` under ACCEPTED | Verify gateway class exists: `kubectl get gatewayclass` |
| **No traffic routing** | curl requests timeout or return errors | Check HTTPRoute configuration and backend service availability |
| **Certificate issues** | HTTPS connections fail | Verify TLS secret exists and certificates are valid |
| **Port conflicts** | NGF service fails to get NodePort | Check for port conflicts: `kubectl get svc -A \| grep <port>` |

### Diagnostic Commands

```bash
# Check all NGF-related resources
kubectl get all -n nginx-gateway

# Check events for errors
kubectl get events -n nginx-gateway --sort-by='.lastTimestamp'

# Check NGINX configuration inside NGF pod
kubectl exec -n nginx-gateway deployment/ngf-nginx-gateway-fabric -- nginx -T

# Test connectivity from within the cluster
kubectl run test-curl --image=curlimages/curl -it --rm -- curl http://ngf-nginx-gateway-fabric.nginx-gateway

# Check resource utilization
kubectl top pods -n nginx-gateway
```

### Uninstallation

To completely remove NGF:

```bash
# Uninstall using Helm
helm uninstall ngf -n nginx-gateway

# Delete the namespace
kubectl delete namespace nginx-gateway

# Remove CRDs (caution: this affects all namespaces)
kubectl delete -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.2.2/deploy/crds.yaml

# Remove Gateway API resources
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.2.2" | kubectl delete -f -
```

## Conclusion

NGINX Gateway Fabric provides a powerful, Kubernetes-native way to manage ingress traffic using the Gateway API standard. The testing methodology using `curl --resolve` with dynamically fetched Node IPs and ports is particularly useful for development and testing environments.

For production deployments, consider:
1. Using LoadBalancer service type
2. Implementing proper TLS termination
3. Setting up monitoring and alerting
4. Configuring appropriate resource limits
5. Implementing backup and disaster recovery procedures

---

*Last Updated: $(date +%Y-%m-%d)*  
*NGINX Gateway Fabric Version: 2.2.2*  
*Kubernetes Version: 1.24+*  

**References:**
- [Official NGF Documentation](https://docs.nginx.com/nginx-gateway-fabric/)
- [Gateway API Specification](https://gateway-api.sigs.k8s.io/)
- [NGINX Plus Documentation](https://docs.nginx.com/nginx-plus/)
