# NGINX Ingress Controller Deployment

This repository contains a complete Kubernetes manifest for deploying an NGINX Ingress Controller using DaemonSet with `hostNetwork` mode, along with an example backend application.

## 📋 Overview

The deployment consists of seven Kubernetes resources that work together to provide ingress capabilities to your cluster:

1. **Namespace** - Isolates ingress-nginx components
2. **ServiceAccount** - Provides identity for the ingress controller
3. **ClusterRole** - Defines RBAC permissions
4. **ClusterRoleBinding** - Binds permissions to the service account
5. **IngressClass** - Defines the ingress controller class
6. **DaemonSet** - Deploys NGINX ingress controller pods on each node
7. **Deployment** - Example backend application for testing

## 🚀 Features

- **DaemonSet Deployment**: Runs one ingress controller pod on each cluster node
- **Host Network Mode**: Uses `hostNetwork: true` for direct node port access
- **RBAC Configuration**: Proper security permissions for Kubernetes API access
- **IngressClass Support**: Uses modern IngressClass API (networking.k8s.io/v1)
- **Example App**: Includes a demo nginx application for testing

## 📁 File Structure

```
ingress-nginx-manifest.yaml
README.md
```

## 🔧 Prerequisites

- Kubernetes cluster (v1.19+ recommended)
- `kubectl` configured with cluster access
- Cluster nodes with ports 80 and 443 available
- Access to pull images from `registry.k8s.io`

## 🛠️ Installation

1. **Apply the manifest**:
   ```bash
   kubectl apply -f ingress-nginx-manifest.yaml
   ```

2. **Verify deployment**:
   ```bash
   # Check all resources are created
   kubectl get all -n ingress-nginx
   
   # Check DaemonSet pods
   kubectl get pods -n ingress-nginx -o wide
   
   # Check the example app
   kubectl get deployment demo-app -n default
   ```

## 🌐 Network Configuration

The ingress controller uses `hostNetwork: true`, which means:
- Pods run directly on the node's network namespace
- No Service LoadBalancer or NodePort is needed
- Controller binds to ports 80 and 443 on each node's IP
- Direct access via any node's IP address

### Ports Exposed:
- **Port 80**: HTTP traffic
- **Port 443**: HTTPS traffic

## 📝 Usage Example

Once deployed, you can create an Ingress resource to route traffic to the demo application:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: demo.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-service
            port:
              number: 80
```

You'll need to create a corresponding Service for the demo app:
```bash
kubectl expose deployment demo-app --port=80 --target-port=80 --name=demo-service -n default
```

## 🔒 Security Notes

1. **Privileged Execution**: The controller runs as root (`runAsUser: 0`) with privilege escalation enabled
2. **Host Network**: Pods have access to the host network stack
3. **Cluster-wide Permissions**: Service account has read access to cluster resources

**Consider for production**:
- Add resource limits and requests
- Implement Pod Security Standards
- Configure TLS certificates
- Add monitoring and logging
- Consider using a dedicated node pool for ingress controllers

## 🧹 Cleanup

To remove all resources:
```bash
kubectl delete -f ingress-nginx-manifest.yaml
```

Or delete individually:
```bash
kubectl delete deployment demo-app -n default
kubectl delete daemonset nginx-ingress-controller -n ingress-nginx
kubectl delete ingressclass nginx
kubectl delete clusterrolebinding nginx-ingress
kubectl delete clusterrole nginx-ingress
kubectl delete serviceaccount nginx-ingress -n ingress-nginx
kubectl delete namespace ingress-nginx
```

## 📊 Verification

Test the ingress controller:
```bash
# Get node IP addresses
kubectl get nodes -o wide

# Test HTTP access (replace NODE_IP with actual node IP)
curl -H "Host: demo.example.com" http://NODE_IP/

# Check controller logs
kubectl logs -n ingress-nginx -l app=nginx-ingress
```

## 🔄 Customization Options

1. **Image Version**: Update `image: registry.k8s.io/ingress-nginx/controller:v1.9.0`
2. **Additional Arguments**: Add to the args section in the DaemonSet
3. **Resource Limits**: Add to the container spec
4. **Node Selectors**: Add to the DaemonSet spec for specific nodes
5. **Tolerations**: Add if running on tainted nodes

## 🆘 Troubleshooting

Common issues and solutions:

1. **Pods not starting**: Check node port availability (80/443)
2. **Image pull errors**: Ensure cluster can access registry.k8s.io
3. **Permission errors**: Verify RBAC resources are properly created
4. **No IP addresses**: Check if nodes have external/internal IPs

Check logs:
```bash
kubectl describe pod -n ingress-nginx -l app=nginx-ingress
kubectl logs -n ingress-nginx -l app=nginx-ingress --tail=50
```

## 📚 References

- [NGINX Ingress Controller Documentation](https://kubernetes.github.io/ingress-nginx/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [DaemonSet Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

## 📄 License

This configuration is provided as-is. Refer to the official NGINX Ingress Controller documentation for detailed configuration options and best practices.
