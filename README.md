<!--
# 🚀 NGINX Plus & Gateway Fabric Installation Suite

A comprehensive toolkit for deploying and managing **NGINX Plus** and **NGINX Gateway Fabric (NGF)** in production Kubernetes environments.
-->

<div align="center">

# 🚀 NGINX Plus & Gateway Fabric Installation Suite

### *Enterprise-Grade Application Delivery Infrastructure*

![NGINX Ecosystem](https://img.shields.io/badge/NGINX-Ecosystem-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![License](https://img.shields.io/badge/License-Guidance_Only-blue?style=for-the-badge)

*A comprehensive toolkit for deploying and managing **NGINX Plus** and **NGINX Gateway Fabric (NGF)** in production Kubernetes environments.*

</div>

---

## 📊 Quick Navigation

| [🏗️ Architecture](#-architecture-overview) | [📚 Guides](#-documentation-guide) | [⚡ Quick Start](#-quick-start) | [🛠️ Tools](#-tools--technologies) |
|------------------------------------------|-----------------------------------|--------------------------------|-----------------------------------|

---

## 🏗️ Architecture Overview

```mermaid
graph TB
    subgraph "Infrastructure Layer"
        A[Ubuntu 22.04 Server] --> B[NGINX Plus Installation]
        B --> C[Kubernetes Cluster<br/>1 Master + 2 Workers]
    end
    
    subgraph "Ingress Solutions"
        C --> D1[<b>Option 1: Nginx Ingress<br/>Open Source</b>]
        C --> D2[<b>Option 2: NGF<br/>Enterprise</b>]
    end
    
    subgraph "Features"
        D1 --> E1[Basic Routing]
        D1 --> E2[SSL/TLS Termination]
        D1 --> E3[Load Balancing]
        
        D2 --> F1[Gateway API]
        D2 --> F2[Advanced Traffic Mgmt]
        D2 --> F3[Metrics & Monitoring]
        D2 --> F4[Security Policies]
    end
    
    E1 --> G[Production Applications]
    E2 --> G
    E3 --> G
    
    F1 --> G
    F2 --> G
    F3 --> G
    F4 --> G
```

---

## 📚 Documentation Guide

| Guide                                            | Purpose                             |
| ------------------------------------------------ | ----------------------------------- |
| **Installing_NGINX_Plus_on_Ubuntu.md**           | NGINX Plus installation & licensing |
| **KubernetesClusterInstallationGuide.md**        | 3-Node Kubernetes cluster           |
| **Kubernetes Nginx Ingress Controller Setup.md** | Open-source ingress                 |
| **NGF-Installation-and-Testing.md**              | NGINX Gateway Fabric                |
| **Advanced_NGF-Configuration-Guide.md**          | Enterprise traffic & security       |

---

## 🆚 Solution Comparison Matrix

| Feature      | Nginx Ingress | NGINX Gateway Fabric |
| ------------ | ------------- | -------------------- |
| License      | Open Source   | Commercial           |
| API          | Ingress       | Gateway API          |
| Traffic Mgmt | Basic         | Advanced             |
| Security     | Basic TLS     | Enterprise policies  |
| Best For     | Dev / Test    | Production           |

---

## 🚀 Quick Start

### Option 1: Open Source

```bash
git clone https://github.com/Abdelrhman2371999/NGINX-Plus-Gateway-Fabric-Installation-Suite.git
cd NGINX-Plus-Gateway-Fabric-Installation-Suite
kubectl get all -n ingress-nginx
```

### Option 2: Enterprise

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm install nginx-gateway nginx-stable/nginx-gateway-fabric
```

---

## 🛠️ Tools & Technologies

| Component     | Technology   |
| ------------- | ------------ |
| OS            | Ubuntu 22.04 |
| Runtime       | containerd   |
| Orchestration | Kubernetes   |
| Ingress       | NGINX / NGF  |
| Load Balancer | MetalLB      |
| Package Mgmt  | Helm         |

---

## 🔍 Diagnostic Commands

```bash
kubectl get nodes
kubectl get pods -A
kubectl get svc -A
kubectl get gatewayclass
```

---

## 🚨 Troubleshooting

<details>
<summary>Pod Pending</summary>

```bash
kubectl describe pod <pod>
kubectl describe node
```
</details>

<details>
<summary>Connection Refused</summary>

```bash
kubectl get svc -n ingress-nginx
sudo ufw allow <NodePort>/tcp
```
</details>

---

## 📈 Implementation Roadmap

```mermaid
timeline
    title NGINX Deployment Timeline
    Week 1 : OS + Kubernetes
    Week 2 : Ingress
    Week 3 : NGF
    Week 4 : Production
```

---

## 🤝 Contributing

```bash
git checkout -b feature/improvement
git commit -am "Improve docs"
git push origin feature/improvement
```

---

## 📖 Resources

* NGINX Docs
* Gateway API
* Kubernetes Ingress
* MetalLB

---

<div align="center">

**Maintained by Abdelrhman Hamed**  
Built with ❤️ for Kubernetes & NGINX

</div>
