# 🏗️ Kubernetes Cluster Architecture — Deep Internals & CKA Troubleshooting

> OS-level, process-level, and pod-level breakdown of how a Kubernetes cluster is born, what runs where, and how CKA tests every failure point.

---

## 🧭 The Big Picture — Two Planes, One Cluster

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE NODE                        │
│                                                                  │
│  [systemd]──► kubelet (daemon)                                   │
│                    │                                             │
│                    ▼  watches /etc/kubernetes/manifests/         │
│         ┌──────────────────────────────────────┐                │
│         │          STATIC PODS                  │                │
│         │  etcd  │  kube-apiserver              │                │
│         │  kube-scheduler  │  kube-controller-  │                │
│         │                    manager            │                │
│         └──────────────────────────────────────┘                │
│                    │ hostNetwork: true (use node IP)             │
│                                                                  │
│  [systemd]──► containerd (CRI daemon)                           │
│  [systemd]──► kubelet (daemon)                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         WORKER NODE                              │
│                                                                  │
│  [systemd]──► kubelet (daemon)  ◄── talks to kube-apiserver    │
│  [systemd]──► containerd (CRI daemon)                           │
│                                                                  │
│  [DaemonSet Pod] kube-proxy  ── programs iptables/IPVS         │
│  [DaemonSet Pod] CNI agent (e.g. calico-node, kube-flannel)    │
│                                                                  │
│  [User Pods scheduled here by scheduler]                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Layer 0 — Pre-Kubernetes: What Must Exist BEFORE kubeadm

These are OS-level prerequisites. kubeadm does NOT install these — you must.

### 1. Container Runtime (containerd)

**What it is:** The low-level engine that actually runs containers. Kubernetes talks to it via the CRI (Container Runtime Interface) gRPC socket.

**Installation:**
```bash
apt-get install -y containerd
# Generate default config
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# IMPORTANT: Set cgroup driver to systemd (must match kubelet)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl enable --now containerd
```

**OS-level process:**
```
systemd
  └── containerd.service          ← /usr/bin/containerd
        └── CRI socket: /run/containerd/containerd.sock
```

**Verify:**
```bash
systemctl status containerd
crictl info                        # talks to containerd via CRI socket
```

---

### 2. kubelet (Node Agent — systemd service)

**What it is:** The most critical Kubernetes process on every node. It:
- Watches `/etc/kubernetes/manifests/` for static pod specs
- Talks to the API server to get pod specs for scheduled workloads
- Tells containerd to start/stop containers via CRI
- Reports node and pod health back to the API server

**Installation:**
```bash
apt-get install -y kubelet kubeadm kubectl
systemctl enable kubelet           # enable but don't start yet — kubeadm will configure it
```

**Key config files written by kubeadm:**
```
/var/lib/kubelet/config.yaml              ← kubelet's runtime config
/var/lib/kubelet/kubeadm-flags.env        ← extra flags (CRI socket path etc.)
/etc/systemd/system/kubelet.service.d/    ← drop-in systemd config
/etc/kubernetes/kubelet.conf              ← kubeconfig for kubelet to auth with API server
```

**OS-level process:**
```
systemd
  └── kubelet.service              ← /usr/bin/kubelet
        ├── --config=/var/lib/kubelet/config.yaml
        ├── --kubeconfig=/etc/kubernetes/kubelet.conf
        ├── --container-runtime-endpoint=unix:///run/containerd/containerd.sock
        └── watches: /etc/kubernetes/manifests/  (staticPodPath)
```

**Verify:**
```bash
systemctl status kubelet
journalctl -u kubelet -f           # live logs
cat /var/lib/kubelet/config.yaml | grep staticPodPath
```

---

### 3. OS Prerequisites

```bash
# Disable swap — Kubernetes WILL NOT start if swap is on
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Load required kernel modules
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# Enable IP forwarding (required for pod networking)
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
```

---

## 🚀 Layer 1 — `kubeadm init`: The Bootstrap Sequence (Step by Step)

When you run `kubeadm init`, it executes these phases in order:

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

### Phase 1: Preflight Checks
```
[preflight] Running pre-flight checks
  - Is swap disabled?
  - Is containerd running?
  - Are required ports available? (6443, 2379, 2380, 10250, 10259, 10257)
  - Is the user root?
  - Does the node have enough CPU/RAM?
```

**Required ports (memorise for CKA):**
| Port | Component | Direction |
|------|-----------|-----------|
| 6443 | kube-apiserver | Inbound (all → control plane) |
| 2379–2380 | etcd | Control plane internal |
| 10250 | kubelet API | Control plane → worker nodes |
| 10259 | kube-scheduler | Internal |
| 10257 | kube-controller-manager | Internal |
| 30000–32767 | NodePort Services | External → worker nodes |

---

### Phase 2: Generate TLS Certificates

**Written to:** `/etc/kubernetes/pki/`

```
/etc/kubernetes/pki/
├── ca.crt / ca.key                        ← Cluster CA (root of trust)
├── apiserver.crt / apiserver.key          ← API server TLS cert
├── apiserver-kubelet-client.crt/.key      ← API server → kubelet auth
├── apiserver-etcd-client.crt/.key         ← API server → etcd auth
├── front-proxy-ca.crt / .key             ← Aggregation layer CA
├── front-proxy-client.crt / .key
├── sa.pub / sa.key                        ← Service Account signing keys
└── etcd/
    ├── ca.crt / ca.key                    ← etcd CA
    ├── server.crt / server.key            ← etcd server TLS
    ├── peer.crt / peer.key                ← etcd peer TLS
    └── healthcheck-client.crt / .key
```

```bash
# Check certificate expiry — critical for CKA
kubeadm certs check-expiration

# Renew all certs
kubeadm certs renew all
```

---

### Phase 3: Generate Kubeconfig Files

**Written to:** `/etc/kubernetes/`

```
/etc/kubernetes/
├── admin.conf             ← kubectl config for cluster admin (copy to ~/.kube/config)
├── kubelet.conf           ← kubelet's identity to talk to API server
├── controller-manager.conf ← controller-manager's kubeconfig
└── scheduler.conf         ← scheduler's kubeconfig
```

```bash
# After kubeadm init — set up kubectl access
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### Phase 4: Write Static Pod Manifests → kubelet Starts Control Plane

This is the **heart of the bootstrap**. kubeadm writes YAML files to `/etc/kubernetes/manifests/`. The kubelet — which is already running as a systemd service — watches this directory and **immediately starts these as static pods**.

```
/etc/kubernetes/manifests/
├── etcd.yaml                      ← written first
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
└── kube-scheduler.yaml
```

**Startup order (dependency chain):**
```
kubelet (systemd) starts watching /etc/kubernetes/manifests/
    │
    ├─► starts etcd container       (no dependencies)
    │       │
    │       └─► etcd listening on :2379 (localhost) and :2380 (peers)
    │
    ├─► starts kube-apiserver       (depends on etcd being ready)
    │       │
    │       ├─► connects to etcd via /etc/kubernetes/pki/etcd/...
    │       └─► exposes HTTPS API on :6443
    │
    ├─► starts kube-controller-manager  (depends on API server)
    │       │
    │       └─► watches API server for resource state changes
    │
    └─► starts kube-scheduler           (depends on API server)
            │
            └─► watches API server for unscheduled pods
```

**Key property: `hostNetwork: true`**
All static pods run with the node's IP (not a pod IP). This is why you see:
```
etcd-controlplane           # IP = node IP (e.g. 192.168.1.10)
kube-apiserver-controlplane # IP = node IP
```

**These are NOT regular pods.** They are called **mirror pods** — you can see them with `kubectl get pods -n kube-system` but you cannot delete or edit them via kubectl. The source of truth is the YAML file on disk.

```bash
# See static pod manifests
ls /etc/kubernetes/manifests/

# Edit a static pod (kubelet auto-restarts it)
vi /etc/kubernetes/manifests/kube-apiserver.yaml

# See mirror pods in kube-system
kubectl get pods -n kube-system
# NAME                                    READY   STATUS    RESTARTS   AGE
# etcd-controlplane                       1/1     Running   0          5m
# kube-apiserver-controlplane             1/1     Running   0          5m
# kube-controller-manager-controlplane    1/1     Running   0          5m
# kube-scheduler-controlplane             1/1     Running   0          5m
```

---

### Phase 5: kubeadm init phase addon — Dynamic Pods via API Server

Once the API server is healthy, kubeadm creates these **via the Kubernetes API** (not static pods):

#### kube-proxy — DaemonSet
```bash
# kube-proxy is deployed as a DaemonSet in kube-system
kubectl get daemonset -n kube-system kube-proxy

# It runs on EVERY node (control plane + workers)
# Programs iptables/IPVS rules for Service networking
# Config stored in ConfigMap:
kubectl get cm -n kube-system kube-proxy -o yaml
```

#### CoreDNS — Deployment (2 replicas)
```bash
# CoreDNS is a Deployment (not DaemonSet)
kubectl get deployment -n kube-system coredns

# Service named "kube-dns" for backwards compatibility
kubectl get service -n kube-system kube-dns
# NAME       TYPE        CLUSTER-IP   PORT(S)
# kube-dns   ClusterIP   10.96.0.10   53/UDP, 53/TCP

# Config in ConfigMap:
kubectl get cm -n kube-system coredns -o yaml

# CoreDNS pods are PENDING until CNI is installed
# They need pod networking to get an IP
```

---

### Phase 6: CNI Plugin — MUST Install Manually

kubeadm does NOT install networking. CoreDNS will stay **Pending** until you do this.

```bash
# Option A: Calico (most common in CKA)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Option B: Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Option C: Weave
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

**After CNI installs:**
```bash
# CNI agent runs as a DaemonSet on every node
kubectl get daemonset -n kube-system          # e.g. calico-node or kube-flannel-ds

# Now CoreDNS gets pod IPs and starts Running
kubectl get pods -n kube-system
# coredns-xxx   1/1   Running  ← now green
```

---

### Phase 7: Worker Node Joins (`kubeadm join`)

```bash
# On worker node — run the token command from kubeadm init output
kubeadm join 192.168.1.10:6443 \
  --token abc123.xyz456 \
  --discovery-token-ca-cert-hash sha256:...

# What this does:
# 1. kubeadm writes kubelet config: /var/lib/kubelet/config.yaml
# 2. kubeadm writes kubelet kubeconfig: /etc/kubernetes/kubelet.conf
# 3. Runs: systemctl daemon-reload && systemctl restart kubelet
# 4. kubelet registers itself with the API server
# 5. kube-proxy DaemonSet automatically schedules a pod on the new node
# 6. CNI DaemonSet automatically schedules a pod on the new node
```

---

## 📦 What's in kube-system After a Full Cluster Is Up?

```bash
kubectl get all -n kube-system
```

| Resource | Name | Type | What it is |
|----------|------|------|-----------|
| Pod | `etcd-<node>` | Static Pod | Key-value store — all cluster state |
| Pod | `kube-apiserver-<node>` | Static Pod | REST API gateway, auth/authz |
| Pod | `kube-controller-manager-<node>` | Static Pod | Reconciliation loops (deployment, node, SA controllers) |
| Pod | `kube-scheduler-<node>` | Static Pod | Assigns pods to nodes |
| DaemonSet | `kube-proxy` | DaemonSet | iptables/IPVS rules for Services — runs on ALL nodes |
| DaemonSet | `calico-node` / `kube-flannel-ds` | DaemonSet | Pod networking — runs on ALL nodes |
| Deployment | `coredns` | Deployment (2 replicas) | Cluster DNS resolution |
| Deployment | `calico-kube-controllers` | Deployment | CNI controller (Calico only) |
| Service | `kube-dns` | ClusterIP | DNS service IP (always 10.96.0.10 by default) |
| ConfigMap | `kubeadm-config` | ConfigMap | Stores kubeadm ClusterConfiguration |
| ConfigMap | `kube-proxy` | ConfigMap | kube-proxy configuration |
| ConfigMap | `coredns` | ConfigMap | CoreDNS Corefile config |

---

## 🔬 Component Deep Dives

### etcd — The Brain's Memory

**What it stores:**
- All Kubernetes objects (pods, deployments, services, secrets, configmaps...)
- Cluster state (node status, lease objects)
- RBAC policies, ServiceAccounts

**How it runs:**
```
Static Pod: /etc/kubernetes/manifests/etcd.yaml
Process: /usr/local/bin/etcd
Ports:
  :2379 → client requests (API server talks here)
  :2380 → peer communication (HA only)
Data dir: /var/lib/etcd/
```

**Critical files:**
```
/etc/kubernetes/pki/etcd/ca.crt          ← etcd CA
/etc/kubernetes/pki/etcd/server.crt      ← server cert
/etc/kubernetes/pki/etcd/server.key      ← server key
/etc/kubernetes/pki/etcd/peer.crt        ← peer cert (HA)
/var/lib/etcd/                           ← data directory
```

**Health check:**
```bash
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# List all keys (shows everything stored)
ETCDCTL_API=3 etcdctl get / --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

### kube-apiserver — The Gatekeeper

**Every kubectl command goes through here.** It is the only component that talks to etcd.

```
Static Pod: /etc/kubernetes/manifests/kube-apiserver.yaml
Process: /usr/bin/kube-apiserver
Port: :6443 (HTTPS, requires TLS)
```

**Key flags in kube-apiserver.yaml:**
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.1.10        # node IP
    - --etcd-servers=https://127.0.0.1:2379  # etcd location
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --authorization-mode=Node,RBAC         # auth mode
    - --service-cluster-ip-range=10.96.0.0/12
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
```

**Request flow for `kubectl get pods`:**
```
kubectl → HTTPS :6443 → kube-apiserver
                             │
                   ┌─────────┼─────────┐
                   ▼         ▼         ▼
              Authentication  Authorization  Admission
              (TLS cert/token)  (RBAC)      (mutating/validating webhooks)
                             │
                             ▼
                           etcd  (read/write)
                             │
                             ▼
                        Response to kubectl
```

---

### kube-scheduler — The Matchmaker

**What it does:** Watches for pods with `nodeName: ""` (unscheduled) and assigns them to a node.

```
Static Pod: /etc/kubernetes/manifests/kube-scheduler.yaml
Process: /usr/bin/kube-scheduler
Port: :10259 (HTTPS, metrics + health)
```

**Scheduling decision factors:**
1. Filter — eliminate nodes that can't run the pod (resources, taints, affinity)
2. Score — rank remaining nodes (most available resources, spread, etc.)
3. Bind — write `nodeName` to the pod spec in etcd via API server

**If scheduler is down:**
```bash
# New pods get stuck in Pending state
kubectl get pods  # STATUS: Pending
kubectl describe pod <pod>  # Events: no events (scheduler not picking it up)
```

---

### kube-controller-manager — The Reconciler

**What it does:** Runs ~30 controllers in a single binary. Each controller watches API server state and reconciles to desired state.

```
Static Pod: /etc/kubernetes/manifests/kube-controller-manager.yaml
Process: /usr/bin/kube-controller-manager
Port: :10257 (HTTPS, metrics + health)
```

**Key controllers running inside:**
| Controller | What it does |
|-----------|-------------|
| Deployment Controller | Creates/manages ReplicaSets |
| ReplicaSet Controller | Ensures desired replica count |
| Node Controller | Marks nodes NotReady, evicts pods |
| ServiceAccount Controller | Creates default SA in new namespaces |
| Namespace Controller | Cleans up resources in deleted namespaces |
| Job Controller | Manages Job completion |
| EndpointSlice Controller | Keeps Service → Pod mapping updated |

**If controller-manager is down:**
```bash
# Deployments stop working — no new ReplicaSets created
# Nodes go NotReady but pods are NOT evicted (node controller is down)
kubectl get deployment  # stuck, replica count not enforced
```

---

### kubelet — The Node Agent (systemd service)

**THE most important process on every node.** If kubelet dies, the node dies to Kubernetes.

```
OS service: systemd → kubelet.service
Process: /usr/bin/kubelet
Port: :10250 (HTTPS, used by API server to exec/logs/proxy into pods)
Config: /var/lib/kubelet/config.yaml
Kubeconfig: /etc/kubernetes/kubelet.conf
Static pods dir: /etc/kubernetes/manifests/ (on control plane)
```

**What kubelet does:**
1. Watches `/etc/kubernetes/manifests/` → starts static pods
2. Polls API server for pods assigned to this node → starts them
3. Calls containerd via CRI socket to run containers
4. Reports pod/node status back to API server
5. Runs liveness/readiness probes
6. Collects logs/metrics, serves exec/port-forward via :10250

**Key config in /var/lib/kubelet/config.yaml:**
```yaml
kind: KubeletConfiguration
cgroupDriver: systemd               # MUST match containerd
staticPodPath: /etc/kubernetes/manifests
clusterDNS:
  - 10.96.0.10                      # CoreDNS ClusterIP
clusterDomain: cluster.local
```

---

### kube-proxy — The Network Plumber (DaemonSet)

**What it does:** Programs iptables/IPVS rules so that Service ClusterIPs actually route to pod IPs.

```
Type: DaemonSet (runs on EVERY node, including control plane)
Namespace: kube-system
Port: :10256 (health check)
Config: ConfigMap kube-proxy in kube-system
```

**What happens when you create a Service:**
```
kubectl expose deployment nginx --port=80
    │
    ▼
API server creates Service object (ClusterIP: 10.96.100.5)
    │
    ▼
kube-proxy on each node detects the Service
    │
    ▼
kube-proxy programs iptables rule:
  DNAT: 10.96.100.5:80 → pod IP (e.g. 10.244.1.5:80)
    │
    ▼
Traffic to ClusterIP is now redirected to the actual pod
```

---

### CoreDNS — The Name Resolver (Deployment)

**What it does:** Resolves `<service>.<namespace>.svc.cluster.local` to ClusterIP.

```
Type: Deployment (2 replicas by default)
Namespace: kube-system
Service: kube-dns (ClusterIP: 10.96.0.10)
Config: ConfigMap "coredns" in kube-system (Corefile)
```

**How pods find DNS:**
```
Pod starts
    │
    ▼
kubelet injects /etc/resolv.conf into pod:
  nameserver 10.96.0.10       ← CoreDNS ClusterIP
  search default.svc.cluster.local svc.cluster.local cluster.local
    │
    ▼
Pod does: curl http://nginx-svc
    │
    ▼
DNS lookup: nginx-svc.default.svc.cluster.local → 10.96.100.5
    │
    ▼
kube-proxy routes 10.96.100.5 → pod IP
```

**Corefile config:**
```bash
kubectl get cm coredns -n kube-system -o yaml
# Shows: . { errors, health, kubernetes cluster.local, forward . /etc/resolv.conf }
```

---

## 💥 CKA Breaking Scenarios — How the Exam Tests Each Component

The CKA gives you a broken cluster and asks you to find and fix it. Here is every major scenario, the **symptom**, the **cause**, and the **fix**.

---

### 🔴 Scenario 1: kube-apiserver is Down

**Symptom:**
```bash
kubectl get nodes
# Error: The connection to the server 192.168.1.10:6443 was refused
```

**Causes & How to Fix:**

```bash
# STEP 1: You can't use kubectl — SSH to control plane node
ssh controlplane

# STEP 2: Check if kubelet is running
systemctl status kubelet
# If kubelet is dead → that's the root cause, not just apiserver

# STEP 3: Check the static pod manifest for errors
cat /etc/kubernetes/manifests/kube-apiserver.yaml
# Common injected errors:
# - Wrong --etcd-servers URL (typo in IP/port)
# - Wrong --advertise-address
# - Incorrect cert file paths (pointing to non-existent file)
# - Malformed YAML (indentation error)

# STEP 4: Check running containers (use crictl since kubectl is down)
crictl ps | grep apiserver          # should show a running container
crictl ps -a | grep apiserver       # -a shows stopped containers too
crictl logs <container-id>          # read logs

# STEP 5: Fix the manifest file
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# kubelet auto-detects the change and restarts the pod

# STEP 6: Wait ~30 seconds then verify
kubectl get nodes                   # should work again
```

---

### 🔴 Scenario 2: etcd is Down or Corrupt

**Symptom:**
```bash
kubectl get nodes
# Error from server: etcdserver: request timed out
# OR: apiserver responds but all operations hang
```

**Causes & Fix:**

```bash
# Check etcd pod
kubectl get pods -n kube-system etcd-controlplane
crictl ps | grep etcd
crictl logs <etcd-container-id>

# Common injected errors:
# - --data-dir pointing to wrong or empty directory
# - Wrong peer URLs
# - Cert paths incorrect

# Fix manifest
vi /etc/kubernetes/manifests/etcd.yaml

# If asked to RESTORE from backup:
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd.db \
  --data-dir=/var/lib/etcd-restored

# Then update etcd.yaml: BOTH --data-dir AND the hostPath volume mount
# COMMON MISTAKE: updating only one of them — update BOTH:
vi /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd-restored
# Change: volumeMounts.mountPath AND volumes.hostPath.path to /var/lib/etcd-restored
```

---

### 🔴 Scenario 3: kubelet is Down on Worker Node

**Symptom:**
```bash
kubectl get nodes
# NAME         STATUS     ROLES
# worker-1     NotReady   <none>      ← node not reporting in
```

**Fix:**
```bash
# SSH to the broken worker node
ssh worker-1

# Check kubelet
systemctl status kubelet
# Likely: Active: failed OR Active: activating (start-limit-hit)

journalctl -u kubelet --since "5 minutes ago"
# Read the error — common causes:
# - cgroupDriver mismatch: kubelet uses "cgroupfs" but containerd uses "systemd"
# - Wrong --container-runtime-endpoint (wrong CRI socket path)
# - /etc/kubernetes/kubelet.conf has wrong API server address
# - /var/lib/kubelet/config.yaml has bad config

# Fix config
vi /var/lib/kubelet/config.yaml       # fix cgroupDriver or other settings
vi /etc/kubernetes/kubelet.conf        # fix API server URL if wrong

# Restart
systemctl daemon-reload
systemctl restart kubelet

# Verify
kubectl get nodes                      # back to Ready (from control plane)
```

---

### 🔴 Scenario 4: kube-scheduler is Down — Pods Stuck Pending

**Symptom:**
```bash
kubectl get pods
# NAME    READY   STATUS    RESTARTS
# nginx   0/1     Pending   0         ← stuck, never scheduled

kubectl describe pod nginx
# Events: (none)  ← scheduler not picking it up
```

**Fix:**
```bash
# On control plane
kubectl get pods -n kube-system | grep scheduler
# kube-scheduler-controlplane   0/1   CrashLoopBackOff

# Check the manifest
cat /etc/kubernetes/manifests/kube-scheduler.yaml
# Common injected errors:
# - Wrong --kubeconfig path (file doesn't exist)
# - Corrupted YAML

vi /etc/kubernetes/manifests/kube-scheduler.yaml
# Fix the error — kubelet auto-restarts

# Verify
kubectl get pods -n kube-system | grep scheduler   # should be Running
kubectl get pods                                    # nginx now gets scheduled
```

---

### 🔴 Scenario 5: kube-controller-manager Down — Deployments Broken

**Symptom:**
```bash
kubectl create deployment test --image=nginx --replicas=3
kubectl get pods
# (no pods created — ReplicaSet controller not running)

kubectl get replicaset
# NAME   DESIRED   CURRENT   READY
# test   3         0         0       ← current is 0, nothing created
```

**Fix:**
```bash
kubectl get pods -n kube-system | grep controller
# kube-controller-manager-controlplane   0/1   Error

kubectl logs -n kube-system kube-controller-manager-controlplane
# OR use crictl if apiserver is also having trouble

cat /etc/kubernetes/manifests/kube-controller-manager.yaml
# Common injected errors:
# - Wrong --kubeconfig path
# - Wrong --root-ca-file path
# - Cert paths typo

vi /etc/kubernetes/manifests/kube-controller-manager.yaml
# Fix → kubelet auto-restarts
```

---

### 🔴 Scenario 6: CoreDNS Not Working — DNS Failures in Pods

**Symptom:**
```bash
# Inside a pod:
curl http://nginx-svc           # Connection refused or Name not resolved
nslookup nginx-svc              # server can't find nginx-svc: NXDOMAIN
```

**Diagnose:**
```bash
kubectl get pods -n kube-system | grep coredns
# coredns-xxx   0/1   CrashLoopBackOff  ← problem

kubectl logs -n kube-system coredns-<pod-id>
# Common errors:
# - Corefile syntax error
# - Upstream DNS server unreachable (forward . /etc/resolv.conf broken)

# Check the ConfigMap
kubectl describe cm coredns -n kube-system

# Fix Corefile
kubectl edit cm coredns -n kube-system

# Restart CoreDNS pods
kubectl rollout restart deployment coredns -n kube-system

# Also check: is kube-proxy running? (DNS response arrives but routing broken?)
kubectl get daemonset kube-proxy -n kube-system
```

---

### 🔴 Scenario 7: Node Not Joining — `kubeadm join` Fails

**Symptom:**
```bash
kubeadm join 192.168.1.10:6443 --token ... --discovery-token-ca-cert-hash ...
# error: could not find a JoinConfiguration in the provided config file
# OR: x509: certificate has expired or is not yet valid
```

**Fix:**
```bash
# Token may have expired (default TTL: 24 hours)
# On control plane: generate a new token
kubeadm token create --print-join-command

# Or if you lost the CA cert hash:
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'

# Common issue: swap still enabled on worker
swapoff -a

# Common issue: firewall blocking port 6443
ufw allow 6443
```

---

### 🔴 Scenario 8: Wrong kubeconfig / Wrong Context

**Symptom:**
```bash
kubectl get nodes
# Error: the server doesn't have a resource type "nodes"
# OR: Forbidden: nodes is forbidden: User "system:anonymous"
```

**Fix:**
```bash
# Check what context you're in
kubectl config current-context
kubectl config get-contexts

# Switch to the right context (CKA exam: READ the question first!)
kubectl config use-context <context-name>

# If kubeconfig is broken or missing
export KUBECONFIG=/etc/kubernetes/admin.conf
# OR
cp /etc/kubernetes/admin.conf ~/.kube/config
```

---

## 🗂️ Master File Reference — Every Path You Need

### Control Plane Node
```
/etc/kubernetes/
├── admin.conf                         ← kubectl access (copy to ~/.kube/config)
├── kubelet.conf                       ← kubelet auth to API server
├── controller-manager.conf            ← controller-manager auth
├── scheduler.conf                     ← scheduler auth
├── manifests/
│   ├── etcd.yaml                      ← Static pod spec for etcd
│   ├── kube-apiserver.yaml            ← Static pod spec for API server
│   ├── kube-controller-manager.yaml   ← Static pod spec for controller manager
│   └── kube-scheduler.yaml            ← Static pod spec for scheduler
└── pki/
    ├── ca.crt / ca.key
    ├── apiserver.crt / apiserver.key
    ├── apiserver-kubelet-client.crt/.key
    ├── apiserver-etcd-client.crt/.key
    ├── sa.pub / sa.key
    └── etcd/
        ├── ca.crt / ca.key
        ├── server.crt / server.key
        └── peer.crt / peer.key

/var/lib/
├── etcd/                              ← etcd data (back this up!)
└── kubelet/
    ├── config.yaml                    ← kubelet runtime config
    └── kubeadm-flags.env              ← extra kubelet flags

/etc/systemd/system/
└── kubelet.service.d/
    └── 10-kubeadm.conf               ← systemd drop-in for kubelet
```

### Worker Node
```
/etc/kubernetes/
├── kubelet.conf                       ← kubelet auth to API server
└── pki/
    └── ca.crt                         ← cluster CA (for kubelet TLS verify)

/var/lib/kubelet/
├── config.yaml                        ← kubelet config (cgroupDriver etc.)
└── kubeadm-flags.env

/run/containerd/
└── containerd.sock                    ← CRI socket (kubelet talks here)

/etc/cni/net.d/                        ← CNI plugin config files
/opt/cni/bin/                          ← CNI plugin binaries
```

---

## ⚡ Quick Diagnostic Runbook (CKA Exam Cheatsheet)

```bash
# ── STEP 1: Always switch context first ──────────────────────────
kubectl config use-context <context>

# ── STEP 2: What's broken? ───────────────────────────────────────
kubectl get nodes                      # any NotReady?
kubectl get pods -n kube-system        # any control plane pods down?
kubectl get pods -A                    # any app pods broken?

# ── STEP 3: Control plane health ────────────────────────────────
kubectl get pods -n kube-system        # all 4 static pods Running?
ls /etc/kubernetes/manifests/          # all 4 yaml files present?

# ── STEP 4: If kubectl doesn't work → SSH to control plane ──────
ssh controlplane
systemctl status kubelet
crictl ps | grep -E "etcd|apiserver|scheduler|controller"
crictl logs <container-id>

# ── STEP 5: Fix static pod issue ─────────────────────────────────
vi /etc/kubernetes/manifests/<component>.yaml
# kubelet auto-restarts pod within ~20 seconds

# ── STEP 6: Worker node issue ────────────────────────────────────
ssh <worker-node>
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago"
vi /var/lib/kubelet/config.yaml
systemctl daemon-reload && systemctl restart kubelet

# ── STEP 7: Verify fix ───────────────────────────────────────────
kubectl get nodes                      # all Ready?
kubectl get pods -n kube-system        # all Running?
```

---

## 🧠 Mental Model Summary

```
WHAT STARTS KUBERNETES:
  systemd → kubelet → watches /etc/kubernetes/manifests/ → starts etcd, apiserver, scheduler, controller-manager

WHAT KUBEADM CREATES AS STATIC PODS (no API server needed):
  etcd, kube-apiserver, kube-controller-manager, kube-scheduler

WHAT KUBEADM DEPLOYS VIA API (needs API server up):
  kube-proxy (DaemonSet), CoreDNS (Deployment)

WHAT YOU MUST INSTALL MANUALLY:
  CNI plugin (Calico/Flannel/Weave)

WHAT JOINS AUTOMATICALLY AFTER WORKER JOINS:
  kube-proxy pod (DaemonSet), CNI agent pod (DaemonSet)

CKA TROUBLESHOOTING HIERARCHY:
  1. kubectl works? → No → kubelet or apiserver down → SSH in
  2. Pods Pending? → Scheduler down OR no CNI OR node tainted
  3. Deployments not creating pods? → Controller-manager down
  4. DNS broken? → CoreDNS crash OR kube-proxy broken
  5. Node NotReady? → kubelet on that node → SSH and fix
  6. etcd issues? → data-dir wrong OR certs wrong OR restore needed
```

---

*Sources: kubernetes.io official docs, CNCF kubeadm implementation details, devopscube.com architecture guide, techiescamp CKA guide, community CKA exam reports 2025–2026*
