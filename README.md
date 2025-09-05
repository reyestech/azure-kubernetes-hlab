
<p align="center">
  <img src="https://github.com/user-attachments/assets/b1646300-912a-4763-8cb9-0ef8c2c4bedf" width="80%">
</p>

---

# **Azure Kubernetes Mini‑Homelab - AKS with 3 Nodes (In-progress)** 

This document serves as a comprehensive guide for deploying and managing a three-node Kubernetes cluster using Microsoft Azure. The purpose of this lab is to cultivate practical skills in provisioning, container orchestration, networking, and observability—essential competencies for contemporary infrastructure roles.

[![Azure AKS](https://img.shields.io/badge/Azure-AKS-blue?logo=azure-kubernetes-service\&logoColor=white)](https://azure.microsoft.com/)  [![Kubernetes](https://img.shields.io/badge/K8s-1.30-blue?logo=kubernetes)](https://kubernetes.io/) 

---

<p align="center">
  <img src="https://github.com/user-attachments/assets/62155ae2-c00b-426a-852c-e291f24e7e11" alt="syslog-workbook-cluste - Azure Kubernetes GIFs" width="90%" />
</p>


---

## 📜 Introduction

This repository defines a lightweight, **three-node Azure Kubernetes Service (AKS)** lab that simulates datacenter operations in the cloud and can later be mirrored on a **3-node bare-metal cluster**. It focuses on **infrastructure-as-code**, **networking & storage basics**, **observability**, and a small **SOC layer** (Log Analytics + Microsoft Sentinel), so you can demonstrate design, security, and operations end-to-end.

**All necessary scripts and manifests are (and will continue to be) included** to keep the lab reproducible for learners—use it to **replicate the setup exactly** or extend it into an **on-prem homelab** when you have hardware.

### Objectives
- Provision AKS with a managed control plane and dedicated **system/user node pools**, **VNet integration**, and **ingress + load balancing**.
- Manage workloads with **Kubernetes manifests and Helm**, including **HPA** and **Cluster Autoscaler**.
- Configure persistent storage via **CSI drivers** (e.g., **Azure Disk** for RWO and **Azure Files** for multi-writer scenarios).
- Integrate **Azure Monitor / Log Analytics**, enable **Microsoft Sentinel**, and add a sample **Logic Apps** playbook for automated response.

> **Project status:** In progress. Some components may be incomplete or temporarily broken while I iterate. I’m actively adding scripts and manifests/manifests and updating documentation to ensure the full end-to-end scenario is working; known gaps and fixes will be tracked in issues.

**Cost note:** AKS agent nodes, Sentinel/Defender, NAT Gateway, and WAF incur costs—use student credits/small SKUs and scale down when idle.

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

**Rationale** — The implementation of three worker nodes ensures fundamental redundancy while maintaining cost efficiency. Azure manages the control plane, thereby reducing operational overhead.


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

**Rationale** — This section demonstrates the process of executing a stateful, load-balanced workload.

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

**Rationale** — The Ingress component facilitates hostname-based routing, while health probes guarantee that pods are automatically restarted in the event of failure.

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

This roadmap outlines the steps necessary to migrate this cloud-based lab to on-premises hardware that users may already possess.

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

## ✅ **Conclusion**
This lab provides a **portable AKS foundation** with security, observability, and automation that runs in Azure and can be **migrated to a 3-node bare-metal cluster** to simulate a real datacenter infrastructure. It demonstrates the full lifecycle—**deploy → secure → observe → scale → recover**—while highlighting the design trade-offs an Azure Cloud/Security Architect considers. Because the repository includes **ready-to-run scripts and manifests**, learners can quickly reproduce the environment or evolve it into their own on-premises homelab.

![6386134ab603091521e212c6_60e452a399f5cfb803e6efbf_deployment_process](https://github.com/user-attachments/assets/772a3640-1cc9-429d-861e-60b74eca9a9e)

---

# 🚀 **Next Enhancements (Overview)**
### Step 1 — Infrastructure & Migration
- [ ] Expand AKS into a **private, policy-enforced** cluster (ACR, networking guardrails, IaC).
- [ ] Add a **migration runbook** to redeploy the same app on a **3-node bare-metal** cluster.

### Step 2 — Automation
- [ ] Use **GitHub Actions (OIDC)** for full CI/CD: **build → scan → push (ACR) → deploy → smoke test**.
- [ ] Apply **Azure Policy** during deployment for guardrails.

### Step 3 — Resilience
- [ ] Add **Velero backups** and run **chaos tests** (node/pod failures).
- [ ] Document **runbooks** for recovery.

### Step 4 — SOC Layer
- [ ] Forward **AKS + Defender** logs into **Log Analytics + Microsoft Sentinel**.
- [ ] Create **analytic rules** and trigger **Logic Apps playbooks** for automated response.

----

## 📊 Feature Mapping
| Layer / Feature | Azure (AKS)                         | Bare-Metal                                                           |
| --------------- | ----------------------------------- | -------------------------------------------------------------------- |
| Hypervisor      | Azure fabric (managed)              | **Proxmox VE**                                                       |
| Control Plane   | Managed by Azure                    | **PN64 master (kubeadm)**                                            |
| Workers         | 3× `Standard_B2s` VMs               | **NUC-1**, **NUC-2** (kubeadm)                                       |
| Backup/Helper   | n/a                                 | **Raspberry Pi** (backups/ops helper)                                |
| Ingress         | Public LB + Ingress                 | **MetalLB (L2)** + Ingress                                           |
| Storage         | **Azure Disk PVC**                  | **NFS / local-path PVC**                                             |
| Observability   | **Azure Monitor + Log Analytics**   | **Prometheus + Grafana**                                             |
| SOC / Security  | **Microsoft Sentinel** (+ Defender) | **ELK on Minisforum Mini PC ** (Beats/Fluent Bit → LS → ES → Kibana) |

----

## ⚙️ Migration Topology
### Topology Option 1
```
┌─────────────────────────────────────────────┐   Migrate   ┌──────────────────────────────────────────────────┐
│              Azure AKS (managed)            │  ───────▶   │                Bare-Metal K8s Cluster           │
│─────────────────────────────────────────────│             │──────────────────────────────────────────────────│
│ Control Plane: Managed by Azure             │             │ Hypervisor: Proxmox                              │
│=============================================│             │==================================================│    
│ Worker Node 0: Standard_B2s VM              │             │ Control Plane: PN64 (master, kubeadm)            │
│---------------------------------------------│             │--------------------------------------------------│
│ Worker Node 1: Standard_B2s VM              │             │ Worker Node 1: NUC-1 (kubeadm worker)            │
│---------------------------------------------│             │--------------------------------------------------│
│ Worker Node 2: Standard_B2s VM              │             │ Worker Node 2: NUC-2 (kubeadm worker)            │
│---------------------------------------------│             │--------------------------------------------------│       
│ • Workload: NGINX (2 replicas)              │             │ • Backup Node: Raspberry Pi (backups/ops helper) │
│ • Storage: Azure Disk PVC                   │             │ • Workload: NGINX (2 replicas)                   │
│ • Ingress: Public LB + Ingress Controller   │             │ • Storage: NFS / local-path PVC                  │
│ • Observability: Azure Monitor+Log Analytics│             │ • Ingress: MetalLB (L2) + Ingress Controller     │
│ • SOC/Security: **Microsoft Sentinel**      │             │ • Observability: Prometheus + Grafana            │
│ • (AKS + Defender logs → Sentinel analytics)│             │ • SOC/Security: **ELK on Minisforum Nini PC**    │
│                                             │             │ • (Beats/Fluent-Bit → Logstash → Elasticsearch → │
│                                             │             │ • Kibana (optional detection rules/SOAR hooks)   │
└─────────────────────────────────────────────┘             └──────────────────────────────────────────────────┘
```

