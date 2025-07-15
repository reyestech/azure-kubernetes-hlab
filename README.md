
<p align="center">
  <img src="https://github.com/user-attachments/assets/b1646300-912a-4763-8cb9-0ef8c2c4bedf" width="80%">
</p>

# **Azure Kubernetes Mini‑Homelab (3 Nodes)**

This document serves as a comprehensive guide for deploying and managing a three-node Kubernetes cluster using Microsoft Azure. The purpose of this lab is to cultivate practical skills in provisioning, container orchestration, networking, and observability—essential competencies for contemporary infrastructure roles.

[![Azure AKS](https://img.shields.io/badge/Azure-AKS-blue?logo=azure-kubernetes-service\&logoColor=white)](https://azure.microsoft.com/)  [![Kubernetes](https://img.shields.io/badge/K8s-1.30-blue?logo=kubernetes)](https://kubernetes.io/) 

---

<p align="center">
  <img src="https://github.com/user-attachments/assets/1e47cef4-28c7-4aa7-b488-15b21b32d0cc" width="70%">
</p>

## 📜 **Introduction**

This repository outlines the steps required to set up a lightweight three-node Kubernetes environment using Azure Kubernetes Service (AKS). Given that the cluster is entirely hosted in the cloud, individuals can engage in experimental learning safely and cost-effectively, making this resource particularly valuable for learners who have yet to acquire physical hardware. All necessary scripts and manifests are included, enabling either precise replication of the lab or its extension into an on-premises homelab in the future.

**Learning objectives**

1. Deploy an AKS cluster with separate worker nodes
2. Manage workloads with YAML and Helm
3. Configure persistent storage & external load balancing
4. Integrate monitoring and autoscaling

Utilization of Azure’s free tier or student credits further contributes to minimizing costs while individuals refine their Kubernetes expertise.
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

This mini-homelab exemplifies a comprehensive Kubernetes deployment utilizing cost-effective cloud resources. The lab encompasses provisioning, storage, networking, observability, and autoscaling, thereby establishing a solid foundation for further exploration in both cloud and on-premises environments.

**Next Enhancements**

* Replace the demo NGINX app with a production microservice
* Add GitHub Actions for CI/CD automation
* Implement a full observability stack (Prometheus, Grafana, Loki)
* Simulate node failures and document recovery procedures

> *Feel free to fork this repo, adapt it to your environment, and share improvements via pull requests.*

![6386134ab603091521e212c6_60e452a399f5cfb803e6efbf_deployment_process](https://github.com/user-attachments/assets/772a3640-1cc9-429d-861e-60b74eca9a9e)


---

