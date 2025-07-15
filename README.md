# Azure Kubernetes Mini‑Homelab (3‑Node)

> **Hands‑on lab to mimic data‑center operations in the cloud & later on bare metal**
>
---

[![Azure Build](https://img.shields.io/badge/Azure-AKS-blue?logo=azure-kubernetes-service\&logoColor=white)](https://azure.microsoft.com/)  [![Kubernetes](https://img.shields.io/badge/K8s-1.30-blue?logo=kubernetes)](https://kubernetes.io/)  [![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

<p align="center">
  <img src="https://github.com/user-attachments/assets/1e47cef4-28c7-4aa7-b488-15b21b32d0cc" width="70%">
</p>

---

## 📜 **Introduction**  

This project simulates a real-world data center environment by deploying a Kubernetes mini-homelab using Azure Kubernetes Service (AKS). It consists of one control plane node and two worker nodes, forming a lightweight 3-node AKS cluster. The goal of this project is to explore and practice core cloud infrastructure concepts, container orchestration, and cluster management in a structured and cost-effective way.

This lab offers a user-friendly learning experience for students, veterans, and professionals who may not have access to high-end physical hardware. By utilizing Azure’s $100 free student credit, this homelab provides an affordable and flexible foundation for individuals seeking to build their skills toward a bare-metal, gradual, or on-premises homelab setup. Additionally, it serves as early preparation for certifications such as the Kubernetes and Cloud Native Associate (KCNA) by providing guided exposure to YAML, Helm, CLI tools, monitoring, and system troubleshooting—all within a realistic simulation.

## 🏗️ Project Overview

We deploy a lightweight AKS cluster with the following architecture:

1. 1x base node (control plane)
2. 2x worker nodes
3. NGINX workload with a LoadBalancer
4. Persistent Volume for basic stateful workload

The project emphasizes real-world operations, including:

1. YAML manifests
2. PowerShell and CLI setup
3. Monitoring integration

## 🔧 Prerequisites

> ### 💻 **Environment & Access**
>
> * Microsoft Azure subscription (or free \$100 student credit)
> * *(Optional)* Public DNS entry (for ingress setup)
>
> ### 🧰 **Tools & CLIs**
>
> * `Azure CLI` ≥ v2.60
> * `kubectl` ≥ v1.30
> * `Helm` v3 or later
>
> 💡 *Prefer GUI? You can complete this project through the Azure Portal instead of using the CLI. All CLI commands below are fully copy-paste ready.*

## 🛠️ Skills 

> ### ☁️ **Cloud Infrastructure**
>
> * Provisioning AKS clusters
> * Managing control plane and worker nodes
> * Load balancing and persistent volumes
>
> ### ⚙️ **Automation & CLI Tools**
>
> * Azure CLI scripting
> * PowerShell terminal usage
> * Managing clusters with `kubectl`
> * Deploying with Helm charts
>
> ### 🧱 **Kubernetes & Networking**
>
> * Writing and applying YAML manifests
> * Exposing services with LoadBalancers
> * Stateful workload setup using volumes
>
> ### 📊 **Monitoring & Observability**
>
> * Enabling Azure Monitor for AKS
> * Reviewing pod metrics and node health




---
---
---

📜 **Introduction**  

This project was developed as a hands-on way to simulate a real-world data center scenario by building a mini Kubernetes homelab in the cloud using Microsoft Azure. The setup consists of one control plane node and two worker nodes, creating a simple yet powerful configuration with Azure Kubernetes Service (AKS). It serves both as a learning environment and as a stepping stone for transitioning to self-hosted, bare-metal deployments.

The primary goal is to explore infrastructure operations and container orchestration without the need for an expensive on-premise setup. By utilizing Azure's student or free-tier credits, this project provides an affordable and low-risk starting point for IT professionals, veterans, and individuals seeking to transition into cloud or infrastructure roles. It is designed with future homelab builders in mind, providing a repeatable and scalable approach to learning the fundamentals of Kubernetes before investing significant money or resources. If you are starting or preparing for certifications like the KCNA, this project is an excellent starting point.

🏗️ Project Overview

We deploy a lightweight AKS cluster with the following architecture:

1. 1x base node (control plane)
2. 2x worker nodes
3. NGINX workload with a LoadBalancer
4. Persistent Volume for basic stateful workload

The project emphasizes real-world operations, including:
1. YAML manifests
2. PowerShell and CLI setup
3. Monitoring integration

## 🖼 Architecture

```
┌──────────────────────────────────┐
│ Azure AKS (managed controlplane) │
│  • node-0  Standard_B2s          │
│  • node-1  Standard_B2s          │
│  • node-2  Standard_B2s          │
│                                  │
│  ┌──────────────┐   ┌──────────┐ │
│  │  NGINX Pod   │…│ NGINX Pod │ │  ← replicas=2
│  └──────────────┘   └──────────┘ │
│           ▲ PVC (Azure Disk)      │
│           │                       │
│  ┌──────────────────────────────┐ │
│  │ NGINX Ingress + LoadBalancer │ │
│  └──────────────────────────────┘ │
└──────────────────────────────────┘
```

---

## 🔑 Prerequisites

* Azure subscription or Student \$100 credit
* **Azure CLI** ≥ 2.60
* **kubectl** ≥ 1.30
* **Helm 3**
* Optional: public DNS entry for ingress

> 🖥 **If you prefer the portal**, you can create the cluster manually (see FAQ below). The commands here are fully copy‑paste ready.

---

## Step 1 – Cluster Setup

\### What & Why (3–6 lines)
Provision a three‑node AKS cluster (`Standard_B2s`). Azure hosts the control plane for you; you own the worker nodes. Container Insights is enabled so logs/metrics flow to Azure Monitor—mirroring real DC telemetry.

\### Commands

```powershell
# Variables
$rg         = "rg-k8s-homelab"
$location   = "EastUS"
$cluster    = "aks-homelab"
$nodeCount  = 3
$nodeSize   = "Standard_B2s"

az group create --name $rg --location $location
az aks create \
  --resource-group $rg \
  --name $cluster \
  --node-count $nodeCount \
  --node-vm-size $nodeSize \
  --enable-addons monitoring \
  --generate-ssh-keys
az aks get-credentials --resource-group $rg --name $cluster
kubectl get nodes -o wide
```

\### What This Does
Creates three Linux VMs as Kubernetes workers, attaches them to a managed control plane, and pipes metrics to Log Analytics for uptime diagnostics.

---

## Step 2 – Persistent NGINX App

\### Overview
Deploy a highly‑available NGINX service backed by an Azure Disk PVC, then expose it with a public LoadBalancer. Demonstrates stateful workloads + external traffic routing.

\### YAML files

<details>
<summary>pvc.yaml</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-disk
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: managed-premium
  resources:
    requests:
      storage: 2Gi
```

</details>

<details>
<summary>nginx-deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: nginx-app }
spec:
  replicas: 2
  selector: { matchLabels: { app: nginx } }
  template:
    metadata: { labels: { app: nginx } }
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: web
      volumes:
      - name: web
        persistentVolumeClaim:
          claimName: nginx-disk
```

</details>

<details>
<summary>nginx-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata: { name: nginx-lb }
spec:
  type: LoadBalancer
  selector: { app: nginx }
  ports: [ { port: 80 } ]
```

</details>

\### Apply & Test

```bash
kubectl apply -f pvc.yaml
kubectl apply -f nginx-deployment.yaml -f nginx-service.yaml
kubectl get svc nginx-lb -w   # wait for EXTERNAL-IP
```

Visit the IP in your browser.

\### Explanation
Creates 2 NGINX replicas with shared durable storage so data persists across restarts. LoadBalancer assigns a public IP.

---

## Step 3 – Ingress + Health Probes

\### Overview
Implement NGINX Ingress for friendly hostnames and add readiness/liveness probes so Kubernetes can auto‑heal failing pods.

\### Commands

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx
```

Add probes in your deployment YAML:

```yaml
readinessProbe:
  httpGet: { path: /, port: 80 }
  periodSeconds: 5
livenessProbe:
  httpGet: { path: /, port: 80 }
  initialDelaySeconds: 15
  periodSeconds: 20
```

Ingress resource (`nginx-ingress.yaml`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-web
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: nginx.YOURDOMAIN.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-lb
            port:
              number: 80
```

```bash
kubectl apply -f nginx-ingress.yaml
```

\### Explanation
Ingress centralizes routing/TLS; probes let kubelet restart or remove bad pods automatically—mirrors production load‑balancer behavior.

---

## Step 4 – Monitoring + Autoscaling

\### Overview
Enable metrics‑server and a Horizontal Pod Autoscaler (HPA) to scale replicas from 2→5 at 50 % CPU.

\### Commands

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl apply -f hpa.yaml       # file in repo
kubectl run -i --tty load --image=busybox /bin/sh
# inside busybox pod
a while loop to hit site and generate CPU load
while true; do wget -q -O- http://nginx-lb; done
```

Monitor:

```bash
kubectl get hpa -w
```

\### Explanation
Shows self‑healing capacity management—critical for meeting SLA uptime.

---

## Homelab Replication

| Azure                     | Homelab (Proxmox + kubeadm)   |
| ------------------------- | ----------------------------- |
| AKS managed control plane | `kubeadm init` on VM1         |
| 2× worker nodes           | `kubeadm join` on VM2/VM3     |
| Azure Disk PVC            | NFS or local‑path provisioner |
| LoadBalancer              | MetalLB                       |
| Azure Monitor             | Prometheus + Grafana          |

---
---

## 🧹 Cleanup

```bash
az group delete --name rg-k8s-homelab --yes --no-wait
```

✅ **Conclusion**

This AKS mini-homelab project emphasizes essential practical skills in provisioning and managing cloud-based Kubernetes clusters. It showcases real-world capabilities in deploying infrastructure, orchestrating containers, and setting up networks in a scalable and cost-effective environment. With a clear focus on foundational tools and automation practices, this project serves as both a demonstration of skills and a stepping stone toward building a fully self-hosted physical homelab.

As the next step, this lab can be expanded to include:

* Custom Ingress controllers (e.g., NGINX Ingress, Traefik)
* Real-time logging (e.g., Fluentd, ELK)
* CI/CD pipelines using GitHub Actions
* Integration with Azure DevOps or other cloud services

This project is perfect for individuals interested in exploring DevOps, cloud administration, or managing container-based infrastructure in a controlled and cost-effective environment.

![6386134ab603091521e212c6_60e452a399f5cfb803e6efbf_deployment_process](https://github.com/user-attachments/assets/772a3640-1cc9-429d-861e-60b74eca9a9e)

---

