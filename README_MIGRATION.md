# 🚀 EKS Migration & Platform Strategy

This document outlines the migration of our production-style platform from a local **Kind** environment to a fully-managed **Amazon EKS** cluster using **Terraform**.

## 🖼️ Application Dashboards

### Nginx Application
![alt text](image-3.png)

### Grafana Monitoring
![alt text](image-6.png)

### Loki Log Aggregation
![alt text](image-7.png)

---

## 🏗️ Phase 1 — Infrastructure as Code (Terraform)

We designed and provisioned AWS infrastructure using Terraform modules for consistency and scalability.

### VPC Architecture
* **Public-only subnet architecture**: Chosen for cost optimization and learning efficiency.
* **HA Setup**: 2 Availability Zones (`us-east-1a`, `us-east-1b`).
* **Cost Saved**: NAT Gateway removed (saving ~$45/month).
* **ELB Integration**: Public subnets tagged with `kubernetes.io/role/elb = 1` for AWS Load Balancer Controller.
* **Auto-IP**: `map_public_ip_on_launch = true`.

### EKS Cluster Provisioning
* **Version**: Kubernetes 1.29.
* **Security**: API endpoint restricted to current public IP (CIDR-based) for protection.
* **IAM**: IRSA (IAM Roles for Service Accounts) enabled for secure AWS service interaction.

### Managed Node Group
* **Instance Type**: `t3.medium` (Optimized for resource-heavy monitoring stack).
* **Scaling**: Min 1, Max 2, Desired 1.
* **Labels**: `role = "worker"` for easier workload management.

---

## ☸️ Phase 2 — Workload & Metrics

### Application Deployment
The Nginx demo app was migrated with full HPA support:
* **Resources**: 50m CPU / 14Mi Memory requests.
* **Autoscaling**: HPA configured for 50% CPU utilization.
* **Metrics Server**: Manually installed to enable HPA functionality on EKS.

---

## 📊 Phase 3 — Observability Stack (EKS)

We deployed the complete observability suite using Helm, optimized for the EKS environment.

### Prometheus & Grafana
Installed via `kube-prometheus-stack` with sub-path routing:
* **Grafana**: Served at `/grafana`.
* **Config**: `GF_SERVER_ROOT_URL` updated for correct sub-path handling.

### Alertmanager
Configured for sub-path routing:
* **Path**: `/alertmanager`.
* **External URL**: Configured via `alertmanager.alertmanagerSpec.externalUrl` to ensure correct UI links.
* **Rules**: `NginxHighCPU` alert implemented for proactive monitoring.

### Centralized Logging (Loki)
* **Loki Stack**: Installed via Helm (`grafana/loki-stack`).
* **Loki + Promtail**: Aggregating logs from all namespaces for centralized debugging in Grafana.

---

## 🌐 Phase 4 — Ingress & Access Management

We implemented the **AWS Load Balancer Controller** to manage a single entry point for all services.

### Ingress Groups (ALB Sharing)
To save costs and simplify management, we used **Ingress Groups** to share one ALB across multiple namespaces:
* **Group Name**: `monitoring-group`.
* **Routing Strategy**:
    * `/grafana` (Order: 1) -> Grafana Service.
    * `/alertmanager` (Order: 3) -> Alertmanager Service.
    * `/` (Order: 10) -> Nginx Demo App.

---

## 💰 Cost Optimization Strategy

This project was built with a "Cloud Native & Cost Conscious" mindset:
* **NAT Gateway Removal**: Saved ~$45/month by using public subnets.
* **Single Node Setup**: Kept the node group minimal for learning purposes.
* **ALB Consolidation**: Used one ALB for 3+ services instead of one per ingress.
* **No EFS/Route53**: Simplified the architecture to avoid additional fixed costs.

---

## 🏁 Current Project Stage

| Component      | Status    |
| -------------- | --------- |
| VPC            | ✅ Done    |
| EKS Cluster    | ✅ Done    |
| Managed Nodes  | ✅ Done    |
| Metrics Server | ✅ Done    |
| Prometheus     | ✅ Done    |
| Grafana        | ✅ Done    |
| Alertmanager   | ✅ Done    |
| Loki Logging   | ✅ Done    |
| ALB Ingress    | ✅ Done    |

---

## 💡 Key Learnings
- **Sub-path Routing**: Requires careful configuration of both the Ingress (Order) and the Application (Root URL).
- **EKS Defaults**: Unlike local setups, EKS requires manual installation of components like the Metrics Server.
- **Security vs. Cost**: A public-only architecture can be secure if API access is restricted and security groups are tight.
