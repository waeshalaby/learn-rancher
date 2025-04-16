# Rancher 
This guide provides a detailed walkthrough for deploying a highly available Kubernetes cluster with Rancher on top, following a user-provisioned infrastructure (UPI)-style setup similar to OpenShift UPI. It includes VM sizing, F5 load balancer rules, DNS configurations, Kubernetes installation steps using kubeadm, and Rancher deployment.

---

## ✨ What is Rancher?
Rancher is a Kubernetes management platform developed by SUSE. It provides a single pane of glass to manage multiple Kubernetes clusters — whether on-prem, in the cloud, or at the edge. Rancher can provision, import, and manage any CNCF-compliant Kubernetes clusters.

### 🔀 Rancher Architectures: RKE, RKE2, and K3s
Rancher can manage any conformant Kubernetes distribution and even provision its own using:

- **RKE1 (Rancher Kubernetes Engine):**
  - Simpler, Docker-based installer
  - Easy to set up but uses Docker, which is deprecated in Kubernetes

- **RKE2 (Rancher Kubernetes Engine 2):**
  - Successor to RKE1 with built-in security
  - Uses `containerd`, supports CIS hardening
  - Better for production, long-term support

- **K3s:**
  - Lightweight Kubernetes for edge and IoT
  - Can run on ARM devices, minimal resource footprint

### 🐳 Docker vs containerd (and why CRI matters)
- **Docker** was originally used in Kubernetes, but it includes many components not needed by K8s (like CLI, builder, etc.)
- **Kubernetes now uses CRI (Container Runtime Interface)** to talk to runtimes like:
  - `containerd` (used in RKE2, GKE, EKS)
  - `CRI-O` (used in OpenShift)
- Docker doesn’t support CRI natively, which is why `dockershim` was needed. This has been deprecated.
- Today, Kubernetes distros have transitioned to **containerd or CRI-O**, which are lightweight, fast, and purpose-built for Kubernetes

### 🏋 Rancher vs. OpenShift (OCP)
| Feature                      | Rancher                                          | OpenShift (OCP)                              |
|-----------------------------|--------------------------------------------------|----------------------------------------------|
| Purpose                     | Manage multiple Kubernetes clusters              | Kubernetes distribution + PaaS               |
| Kubernetes Included?        | No (runs on existing K8s)                        | Yes (comes with OCP installer)               |
| Installer                   | Helm-based or Docker                             | IPI/UPI automated installer                   |
| App Management              | Helm Charts, Rancher Apps                        | OperatorHub, Templates                        |
| GitOps                      | Fleet (built-in)                                 | ArgoCD (with GitOps operator)                |
| Access Control              | Native RBAC, external auth support               | OpenShift OAuth + RBAC                       |
| Built-in Monitoring         | Yes                                              | Yes                                          |
| Recommended Use Case       | Multi-cluster, hybrid, edge                      | Single or hybrid OpenShift clusters          |
| Best Fit                    | Cloud-neutral, mixed K8s management              | Red Hat ecosystem, compliance-driven orgs    |

...

