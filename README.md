
# Rancher on VMware UPI-Style Installation (Kubernetes HA)

This guide provides a detailed walkthrough for deploying a highly available Kubernetes cluster with Rancher on top, following a user-provisioned infrastructure (UPI)-style setup similar to OpenShift UPI. It includes VM sizing, F5 load balancer rules, DNS configurations, Kubernetes installation steps using kubeadm, and Rancher deployment.

---

## ✨ What is Rancher?

Rancher is a Kubernetes management platform developed by SUSE. It provides a single pane of glass to manage multiple Kubernetes clusters — whether on-prem, in the cloud, or at the edge. Rancher can provision, import, and manage any CNCF-compliant Kubernetes clusters.

### 🔀 Rancher Architectures: RKE, RKE2, and K3s

- **RKE1**: Docker-based, deprecated in modern Kubernetes
- **RKE2**: Successor to RKE1, uses `containerd`, security-focused, CIS-hardened
- **K3s**: Lightweight Kubernetes for edge and IoT, ARM-compatible

---

## 🐳 Docker vs containerd (and why CRI matters)

- Docker = full container suite (CLI, builder, etc.)
- Kubernetes needs just the container runtime
- Kubernetes now uses CRI-compatible runtimes:
  - `containerd`, `CRI-O`
- `dockershim` (bridge for Docker) is deprecated
- `containerd` is the recommended runtime today

---

## 🏋 Rancher vs. OpenShift (OCP)

| Feature                | Rancher                           | OpenShift (OCP)                           |
|------------------------|------------------------------------|-------------------------------------------|
| Kubernetes bundled     | ❌ Uses upstream K8s               | ✅ Comes with K8s                          |
| Install model          | Helm, RKE2, K3s                   | IPI/UPI                                  |
| App management         | Helm, Catalogs, GitOps            | Operators, GitOps                         |
| GitOps                 | Fleet                             | ArgoCD                                   |
| Use case               | Multi-cluster, hybrid, edge       | Full enterprise PaaS                      |

---

## 🧱 VM Sizing

| Node       | Role         | vCPU | RAM  | Disk | OS                  |
|------------|--------------|------|------|------|----------------------|
| master-[1-3]| Control Plane| 2+   | 8GB+ | 40GB | RHEL 8 / Ubuntu 20.04|
| worker-[1-2]| Worker       | 2+   | 8GB+ | 40GB | Same as above        |

---

## 📡 F5 Load Balancer Rules

| DNS Name         | Ports    | Target Nodes  | Purpose             |
|------------------|----------|----------------|----------------------|
| `api.k8s.lab`     | 6443     | master-[1-3]   | K8s API              |
| `rancher.k8s.lab` | 80, 443  | worker-[1-2]   | Rancher UI           |
| `*.apps.k8s.lab`  | 443      | worker-[1-2]   | App ingress          |

---

## 🌐 DNS Records

| Hostname            | Points to (VIP) |
|---------------------|------------------|
| `api.k8s.lab`       | F5 LB for API    |
| `rancher.k8s.lab`   | F5 LB for UI     |
| `*.apps.k8s.lab`    | Wildcard ingress |

---

## 🛠️ Kubernetes Installation (kubeadm HA)

### All Nodes
```bash
hostnamectl set-hostname <hostname>
swapoff -a && sed -i '/swap/d' /etc/fstab
modprobe overlay
modprobe br_netfilter
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system
apt install -y containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
```

### Install K8s Tools
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
apt update && apt install -y kubelet kubeadm kubectl
```

---

### Init Master Node
```bash
kubeadm init --control-plane-endpoint "api.k8s.lab:6443" --upload-certs
```

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
```

---

### Join Other Masters
```bash
kubeadm join api.k8s.lab:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>
```

### Join Workers
```bash
kubeadm join api.k8s.lab:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

### Install CNI
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

## 🐮 Rancher Installation

### Add Helm and Cert-Manager
```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

kubectl apply --validate=false \
  -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

### Install Rancher
```bash
kubectl create namespace cattle-system
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.k8s.lab \
  --set replicas=3 \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@k8s.lab
```

---

## ✅ Final Steps

- Visit `https://rancher.k8s.lab`
- Set admin password
- Rancher auto-imports the cluster
- Add AKS, EKS, GKE, etc.
- Enable GitOps, monitoring, policies

---

## 🔁 TL;DR

- Rancher = Multi-cluster K8s management
- kubeadm = Manual, upstream K8s install
- F5 + DNS pattern = Like OpenShift UPI
- RKE2 or K3s can replace kubeadm
