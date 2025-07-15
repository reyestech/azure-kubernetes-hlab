
# Azure Kubernetes Mini‑Homelab (3 Nodes)

A reproducible guide for deploying and managing a three‑node Kubernetes cluster on **Microsoft Azure**. The lab demonstrates practical skills in provisioning, container orchestration, networking, and observability—key competencies for modern infrastructure roles.

[![Azure AKS](https://img.shields.io/badge/Azure-AKS-blue?logo=azure-kubernetes-service\&logoColor=white)](https://azure.microsoft.com/)  [![Kubernetes](https://img.shields.io/badge/K8s-1.30-blue?logo=kubernetes)](https://kubernetes.io/)  [![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 📜 Introduction

This repository shows how to build a lightweight three‑node Kubernetes environment using **Azure Kubernetes Service (AKS)**. Because the cluster lives entirely in the cloud, you can experiment safely and cost‑effectively—ideal for learners who don’t yet own physical hardware. All scripts and manifests are included so you can replicate the lab exactly, or extend it into an on‑premises homelab later.

**Learning objectives**

1. Deploy an AKS cluster with separate worker nodes
2. Manage workloads with YAML and Helm
3. Configure persistent storage & external load balancing
4. Integrate monitoring and autoscaling

Azure’s free tier or student credit keeps costs low while you sharpen your Kubernetes skills.

---

## 🏗️ Azure Architecture Snapshot

```
┌───────────────────────────────────────────┐
│            Azure AKS (managed)            │
│───────────────────────────────────────────│
│  • node‑0   Standard_B2s  (worker)        │
│  • node‑1   Standard_B2s  (worker)        │
│  • node‑2   Standard_B2s  (worker)        │
│                                           │
│  ┌──────────────┐     ┌──────────────┐    │
│  │  NGINX Pod   │ … │  NGINX Pod   │    │  ← replicas = 2
│  └──────────────┘     └──────────────┘    │
│                ▲ PersistentVolumeClaim    │
│                │ (Azure Managed Disk)     │
│  ┌──────────────────────────────────────┐ │
│  │     Ingress + Public LoadBalancer    │ │
│  └──────────────────────────────────────┘ │
└───────────────────────────────────────────┘
```

| Layer / Feature  | Implementation                       | Purpose                          |
| ---------------- | ------------------------------------ | -------------------------------- |
| Control Plane    | Managed by Azure                     | API server, scheduler, etcd      |
| Worker Nodes (3) |  `Standard_B2s` VMs                  | Run pods and services            |
| Workload Example | NGINX Deployment (2 replicas)        | Demonstrate HA web service       |
| Storage          | Azure Disk PVC                       | Persist data across pod restarts |
| Networking       | LoadBalancer Service + Ingress       | Expose application externally    |
| Observability    | Azure Monitor + Metrics‑Server + HPA | Logs, metrics, auto‑scaling      |

---

## 🔧 Prerequisites

> ### Environment
>
> • Azure subscription (or \$100 student credit)
> • *(Optional)* Public DNS record for ingress
>
> ### Tooling
>
> • **Azure CLI** ≥ 2.60
> • **kubectl** ≥ 1.30
> • **Helm 3**
>
> *All steps can be completed in the Azure Portal if you prefer GUI over CLI.*

---

## 🛠️ Skills Demonstrated

| Category                | Topics Covered                                                         |
| ----------------------- | ---------------------------------------------------------------------- |
| Cloud Infrastructure    | AKS provisioning • Node‑pool management • Disk provisioning            |
| Automation & CLI Tools  | Azure CLI scripting • PowerShell • `kubectl` administration            |
| Kubernetes & Networking | YAML manifests • LoadBalancer services • Ingress • Health probes       |
| Monitoring & Scaling    | Azure Monitor integration • Metrics‑Server • Horizontal Pod Autoscaler |

---

## 1️⃣ Cluster Provisioning

**Rationale** — Three worker nodes provide basic redundancy while keeping costs low. Azure hosts the control plane, reducing operational overhead.

```powershell
$RG       = "rg‑aks‑homelab"
$LOC      = "EastUS"
$CLUSTER  = "aks‑homelab"
$NODES    = 3
$SIZE     = "Standard_B2s"

az group create --name $RG --location $LOC
az aks create `
  --resource-group $RG `
  --name $CLUSTER `
  --node-count $NODES `
  --node-vm-size $SIZE `
  --enable-addons monitoring `
  --generate-ssh-keys
az aks get-credentials --resource-group $RG --name $CLUSTER
kubectl get nodes -o wide
```

*Outcome — Three Linux worker nodes register with the managed control plane, and logs/metrics flow into Azure Monitor.*

---

## 2️⃣ Deploy Persistent NGINX Service

**Rationale** — Shows how to run a stateful, load‑balanced workload.

<details><summary>pvc.yaml</summary>

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

<details><summary>nginx-deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: nginx-deployment }
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
          name: content
      volumes:
      - name: content
        persistentVolumeClaim:
          claimName: nginx-disk
```

</details>

<details><summary>nginx-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata: { name: nginx-service }
spec:
  type: LoadBalancer
  selector: { app: nginx }
  ports:
  - port: 80
```

</details>

```bash
kubectl apply -f pvc.yaml
kubectl apply -f nginx-deployment.yaml -f nginx-service.yaml
kubectl get svc nginx-service -w   # wait for EXTERNAL-IP
```

Navigate to the **EXTERNAL‑IP** in a browser to confirm deployment.

---

## 3️⃣ Ingress & Health Probes

**Rationale** — Ingress provides hostname‑based routing; probes ensure pods are restarted automatically if unhealthy.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx
```

Add readiness/liveness probes in `nginx-deployment.yaml`, then create `nginx-ingress.yaml` pointing your hostname to the NGINX service.

---

## 4️⃣ Monitoring & Autoscaling

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Deploy HPA (`hpa.yaml`):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: nginx-hpa }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Generate load and watch scaling with `kubectl get hpa -w`.

---

## 🏡 Homelab Transition Plan

A roadmap for migrating this cloud lab to on‑premises hardware you already own.

| Azure Feature         | Bare‑Metal Replacement                                 |
| --------------------- | ------------------------------------------------------ |
| Managed Control Plane | **ASUS ExpertCenter PN64** (i5) running `kubeadm init` |
| Worker Nodes (3)      | **Intel NUC11PAKi5 × 2** running `kubeadm join`        |
| Azure Disk PVC        | NFS share on PN64 or `local-path-provisioner`          |
| Azure LoadBalancer    | **MetalLB** (Layer‑2 mode)                             |
| Azure Monitor         | **Prometheus + Grafana** via Helm                      |

> **Hypervisors:** Proxmox VE (preferred) or VMware ESXi Free

### Bare‑Metal Cluster Diagram

```
┌───────────────────────────────────────────┐
│           Bare‑Metal K8s Cluster          │
│───────────────────────────────────────────│
│  • PN64  (master)  – kubeadm control‑plane│
│  • NUC‑1 (worker)  – kubeadm node         │
│  • NUC‑2 (worker)  – kubeadm node         │
│                                           │
│  NFS / local‑path PVC • MetalLB • Ingress │
└───────────────────────────────────────────┘
```

---

## ✅ Conclusion

This mini‑homelab demonstrates end‑to‑end Kubernetes deployment using cost‑effective cloud resources. The lab covers provisioning, storage, networking, observability, and autoscaling—forming a solid foundation for deeper exploration in both cloud and bare‑metal environments.

**Next Enhancements**

* Replace the demo NGINX app with a production microservice
* Add GitHub Actions for CI/CD automation
* Implement a full observability stack (Prometheus, Grafana, Loki)
* Simulate node failures and document recovery procedures

> *Feel free to fork this repo, adapt it to your own environment, and share improvements via pull requests.*


---
---
---
---

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

## 🏗️ **Project Overview**  

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

\### What This Does
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

