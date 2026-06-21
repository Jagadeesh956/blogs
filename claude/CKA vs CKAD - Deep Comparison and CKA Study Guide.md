# ⚙️ CKA vs CKAD — Deep Comparison & CKA Prep Guide

> Written for someone who has already cleared CKAD. Focuses on what's **new and different** in CKA, the level of depth required, and the best resources to clear it.

---

## 🧭 The Core Mental Model

| | CKAD | CKA |
|-|------|-----|
| **You are a...** | Developer shipping apps onto a cluster | Administrator keeping the cluster alive |
| **Cluster is...** | Already running — not your problem | Your responsibility to install, upgrade, fix |
| **Your job is...** | Deploy, configure, debug your app | Bootstrap, secure, network, and recover the cluster |
| **Analogy** | Tenant in a building | Building superintendent |

> **CKAD assumes the cluster exists. CKA assumes you have to build and maintain it.**

---

## 📊 Exam Structure — Side by Side

| Parameter | CKAD | CKA |
|-----------|------|-----|
| **Duration** | 2 hours | 2 hours |
| **No. of Tasks** | 15–20 tasks | 15–20 tasks |
| **Format** | Performance-based (terminal only) | Performance-based (terminal only) |
| **Pass Score** | 66% | 66% |
| **Cost** | $445 (1 free retake) | $445 (1 free retake) |
| **Validity** | 2 years | 2 years |
| **Kubernetes Version** | v1.32–v1.34 | v1.34 |
| **Pass Rate** | ~60–65% | ~55–60% |
| **Simulator included** | killer.sh (2 sessions) | killer.sh (2 sessions) |
| **Open book?** | Yes — kubernetes.io only | Yes — kubernetes.io only |
| **Difficulty** | 🟡 Moderate | 🔴 Moderate-Hard |

---

## 🗺️ Domain Breakdown — CKAD vs CKA

### CKAD Domains (What You Already Know)

| Domain | Weight | Focus |
|--------|--------|-------|
| Application Environment, Configuration & Security | **25%** | ConfigMaps, Secrets, SecurityContext, ServiceAccounts, ResourceQuotas |
| Application Design and Build | **20%** | Multi-container pods, init containers, jobs, CronJobs, Dockerfile, images |
| Application Deployment | **20%** | Deployments, rolling updates, rollbacks, Helm, Kustomize |
| Services and Networking | **20%** | ClusterIP, NodePort, Ingress, NetworkPolicies |
| Application Observability and Maintenance | **15%** | Liveness/Readiness probes, logs, metrics, API deprecations |

---

### CKA Domains (What You Need to Learn)

| Domain | Weight | Focus |
|--------|--------|-------|
| **Troubleshooting** | **30%** | 🔴 Largest domain — cluster-level debugging |
| **Cluster Architecture, Installation & Configuration** | **25%** | kubeadm, etcd, control plane, RBAC, upgrades |
| **Services & Networking** | **20%** | CNI plugins, CoreDNS, Ingress, NetworkPolicies |
| **Workloads & Scheduling** | **15%** | DaemonSets, static pods, taints, affinity, resource limits |
| **Storage** | **10%** | StorageClasses, PVs, PVCs, dynamic provisioning |

> 💡 **Cluster Architecture + Troubleshooting = 55% of the CKA exam.** These two domains alone determine whether you pass.

---

## 🔍 Topic-by-Topic: What's New in CKA vs CKAD

### 🆕 Topics EXCLUSIVE to CKA (Not in CKAD at all)

#### 1. Cluster Architecture & Bootstrap (25%)
- **kubeadm install** — bootstrapping a multi-node cluster from scratch
  - `kubeadm init`, `kubeadm join`, installing CNI plugin after init
- **Control plane components** — deep understanding of each:
  - `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `etcd`
  - How to check their status via `kubectl get pods -n kube-system` and `systemctl`
- **etcd backup and restore** — one of the most tested CKA tasks
```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-restore
# Then update etcd static pod manifest: --data-dir=/var/lib/etcd-restore
```
- **Cluster upgrades with kubeadm**
  - `kubeadm upgrade apply v1.3x.0`
  - Drain node → upgrade kubelet/kubectl → uncordon node
- **High Availability clusters** — stacked vs external etcd topology
- **Static pods** — managed by kubelet directly via `/etc/kubernetes/manifests/`
- **TLS certificates** — `kubeadm certs check-expiration`, certificate rotation

#### 2. Troubleshooting (30%)
The hardest and most unique part of CKA. You debug at infrastructure level, not app level.

**Control plane failures:**
```bash
kubectl get pods -n kube-system
kubectl describe pod kube-apiserver-<node> -n kube-system
crictl logs <container-id>
journalctl -u kube-apiserver
# Check and fix static pod manifests
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Worker node failures:**
```bash
kubectl describe node <node-name>
ssh <node>
systemctl status kubelet
journalctl -u kubelet -f
# Fix and restart
systemctl restart kubelet
```

**CoreDNS / networking failures:**
```bash
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system coredns-<pod>
kubectl edit cm coredns -n kube-system
```

#### 3. Node Management
- `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`
- `kubectl cordon <node>` / `kubectl uncordon <node>`
- Node conditions: `Ready`, `MemoryPressure`, `DiskPressure`, `NetworkUnavailable`

#### 4. Networking — Deeper Than CKAD
- **CNI plugins** — understanding and installing (Calico, Flannel, Weave)
- **CoreDNS** — configuration, DNS resolution patterns: `<svc>.<ns>.svc.cluster.local`
- NetworkPolicies tested more heavily than in CKAD

#### 5. Storage — Deeper Than CKAD
- **StorageClasses** — defining dynamic provisioners
- **PersistentVolumes** — manual creation, access modes, reclaim policies
- **Volume types** — hostPath, emptyDir, NFS for static PVs

---

### ♻️ Topics Shared Between CKAD and CKA (But Tested Differently)

| Topic | CKAD Depth | CKA Depth |
|-------|-----------|-----------|
| **RBAC** | Use ServiceAccounts in apps | Create Roles, ClusterRoles, troubleshoot access |
| **Services** | Expose apps via ClusterIP/NodePort | Also debug kube-proxy, iptables/IPVS |
| **NetworkPolicies** | Apply policies to apps | Heavier testing, debug DNS through policies |
| **Scheduling** | Basic resource limits | Taints, tolerations, node affinity, DaemonSets, static pods |
| **PV/PVC** | Mount volumes in apps | Create PVs, debug binding, StorageClasses |

---

## 🧠 Effort Map (As a CKAD Holder)

| Area | CKAD Foundation | CKA Additional Effort |
|------|----------------|----------------------|
| Pod creation, Deployments | ✅ Strong | Minimal |
| Services, Ingress | ✅ Strong | Add CNI + CoreDNS debugging |
| RBAC | ✅ Moderate | Add ClusterRole, `kubectl auth can-i` troubleshooting |
| Storage | ✅ Basic PVC usage | Add StorageClass creation, PV binding debug |
| NetworkPolicies | ✅ Can write them | Same — just more exam weight |
| **etcd backup/restore** | ❌ Not in CKAD | Must learn from scratch — high exam weight |
| **kubeadm / cluster bootstrap** | ❌ Not in CKAD | Must learn from scratch — 25% of exam |
| **Control plane troubleshooting** | ❌ Not in CKAD | Must learn from scratch — part of 30% |
| **Node troubleshooting (kubelet)** | ❌ Not in CKAD | Must learn from scratch — part of 30% |
| **Cluster upgrades** | ❌ Not in CKAD | Must learn from scratch |

> **Estimate:** If CKAD took you X weeks, CKA needs ~0.6X additional weeks given your foundation.

---

## 🔑 High-Value Tasks to Practice Religiously

### 1. etcd Backup & Restore
```bash
export ETCDCTL_API=3

# Save snapshot
etcdctl snapshot save /opt/backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
etcdctl snapshot status /opt/backup/etcd.db --write-out=table

# Restore
etcdctl snapshot restore /opt/backup/etcd.db --data-dir=/var/lib/etcd-new
# Update /etc/kubernetes/manifests/etcd.yaml: --data-dir and volumeMount path
```

### 2. Cluster Upgrade (kubeadm)
```bash
# Control plane
apt-get install -y kubeadm=1.3x.0-00
kubeadm upgrade plan
kubeadm upgrade apply v1.3x.0
apt-get install -y kubelet=1.3x.0-00 kubectl=1.3x.0-00
systemctl daemon-reload && systemctl restart kubelet

# Worker nodes
kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data
# SSH into worker, upgrade kubeadm + kubelet + kubectl
kubectl uncordon <worker>
```

### 3. Node Troubleshooting
```bash
kubectl get nodes                        # Identify not-ready node
ssh <node>
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago"
systemctl enable --now kubelet           # If not running
```

### 4. RBAC — Create Role and Bind
```bash
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods -n dev

kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --user=john -n dev

kubectl auth can-i get pods --as john -n dev  # Verify
```

### 5. PersistentVolume + StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/pv
  storageClassName: fast
```

---

## 📚 Best Resources for CKA (Post-CKAD)

### 🥇 Must Use

| Resource | Notes |
|----------|-------|
| **Mumshad Mannambeth — CKA (Udemy / KodeKloud)** | Gold standard. Labs for etcd and kubeadm are unmatched. Wait for Udemy sale ($10–15). |
| **killer.sh** (free with exam purchase) | Harder than real exam. Two sessions included. Use 1st mid-prep, 2nd two days before. |
| **kubernetes.io/docs** | Your open book during the exam. Practice navigating it fast. |
| **KodeKloud Lightning Labs + Mock Exams** | Since you have CKAD, skip intro and jump straight to these. |

### 🥈 Highly Recommended

| Resource                                         | Notes                                                                   |
| ------------------------------------------------ | ----------------------------------------------------------------------- |
| **killercoda.com**                               | Free browser-based K8s playground. Great for etcd and kubeadm.          |
| **techiescamp/cka-certification-guide** (GitHub) | Community-maintained study notes + kubectl cheat sheets.                |
| **devopscube.com CKA study guide**               | Detailed and kept current with each K8s version.                        |
| **Vagrant + VirtualBox**                         | Best way to practice real kubeadm cluster setup — painful but worth it. |

### 🥉 Supplemental

| Resource | Notes |
|----------|-------|
| **r/kubernetes + r/cka** (Reddit) | Check before scheduling — latest exam changes and community tips. |
| **Kubernetes the Hard Way** (Kelsey Hightower) | Overkill for exam but builds deep control plane understanding. |

---

## ⏱️ 4-Week Study Plan (For CKAD Holders)

### Week 1 — Cluster Architecture
- [ ] Install kubeadm cluster from scratch (do this 3+ times)
- [ ] Understand every control plane component and its flags
- [ ] Master etcd backup and restore (practice daily)
- [ ] Learn static pods — what they are, where they live (`/etc/kubernetes/manifests/`)
- [ ] Practice cluster version upgrade end-to-end

### Week 2 — Troubleshooting Mastery
- [ ] Break your cluster intentionally and fix it
- [ ] API server down → fix via static pod manifest
- [ ] kubelet not running on worker → SSH and fix
- [ ] CoreDNS failing → diagnose and fix
- [ ] etcd unhealthy → restore from snapshot
- [ ] `kubectl` not connecting → kubeconfig issues

### Week 3 — Networking, Storage, Gaps
- [ ] CNI plugin deep dive — how Calico/Flannel work
- [ ] StorageClass + dynamic provisioning
- [ ] Manual PersistentVolume creation and PVC binding debug
- [ ] RBAC edge cases — ClusterRole vs Role, `kubectl auth can-i`
- [ ] Node drain/cordon/uncordon workflows

### Week 4 — Mock Exams + Speed Drills
- [ ] killer.sh Session 1 (full 2-hour simulation)
- [ ] Review every wrong answer, drill those areas
- [ ] KodeKloud Mock Exams 1–3
- [ ] Speed target: etcd backup in under 3 minutes
- [ ] killer.sh Session 2 (2 days before exam)

---

## 💡 Exam Day Tips

### Set These Aliases First Thing
```bash
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"
```

### Vim Setup
```bash
# ~/.vimrc
set nu
set expandtab
set shiftwidth=2
set tabstop=2
set autoindent
```

### Context Switching — Do This Before Every Question
```bash
kubectl config use-context <context-name>
# Printed at top of each question — never skip this step
```

### Time Management
- 17 questions × ~7 min avg = fits in 2 hours
- Questions show point values — **start high-value questions first**
- Flag and skip if stuck — don't lose 10 min on a 2% question
- Troubleshooting questions take longest — budget 10–15 min each

### Docs to Bookmark During Exam
- etcd backup → search "etcd backup"
- kubeadm upgrade → search "upgrading kubeadm clusters"
- RBAC → Tasks → Configure RBAC
- NetworkPolicy → Concepts → Network Policies
- StorageClass → Concepts → Storage → Storage Classes

---

## 🎯 What Separates Passers from Failers

| Passers | Failers |
|---------|---------|
| Practice etcd backup/restore 20+ times | Read about it but never type the commands |
| Build a real kubeadm cluster from scratch | Only use managed K8s (EKS, GKE) |
| SSH into nodes and debug kubelet | Stay at `kubectl` level only |
| Use killer.sh seriously | Skip it or treat it casually |
| Switch cluster context before every question | Jump in and work on wrong cluster |
| Finish with 10+ minutes to review | Run out of time |

---

## 🚀 The Path Forward: Kubestronaut

Once CKA is cleared, you're 2/5 to the **Kubestronaut** title (CNCF's elite badge):

| Cert | Status | Focus |
|------|--------|-------|
| CKAD | ✅ Cleared | App development |
| CKA | 🎯 Next | Cluster administration |
| CKS | ⬜ After CKA | Kubernetes security (requires active CKA) |
| KCNA | ⬜ Optional | Cloud Native associate (theory-based) |
| KCSA | ⬜ Optional | Cloud Native security associate |

> CKS requires an **active CKA** — clearing CKA unlocks the full security track.

---

*Sources: CNCF official curriculum, Linux Foundation exam portal, killer.sh, KodeKloud, devopscube.com, community exam reports 2025–2026*
