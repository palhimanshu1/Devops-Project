# ☸️ Kubernetes Cluster Setup on Virtual Machines

A complete step-by-step guide to deploy a production-ready Kubernetes cluster using `kubeadm` on bare-metal/virtual machines with **Containerd** as the container runtime and **Calico** as the CNI plugin.

---

## 📋 Cluster Info

| Role       | Hostname      | IP Address       |
|------------|---------------|------------------|
| Master     | k8s-master    | 192.168.228.17   |
| Worker 1   | k8s-worker1   | 192.168.228.19   |
| Worker 2   | k8s-worker2   | 192.168.228.22   |

- **OS:** RHEL / Rocky / Alma / CentOS 8+
- **Credentials:** `root` / `k8s-admin`
- **Kubernetes Version:** v1.29
- **CNI:** Calico v3.27.0

---

## 🗺️ Setup Overview

```
Phase 0 → Clean VMs
Phase 1 → OS Pre-configuration (All Nodes)
Phase 2 → Install Containerd (All Nodes)
Phase 3 → Install Kubernetes Components (All Nodes)
Phase 4 → Initialize Cluster (Master Only)
Phase 5 → Install Calico CNI (Master Only)
Phase 6 → Join Workers
Phase 7 → Verify Cluster
```

---

## 🧹 Phase 0 — Clean VMs (All Nodes)

Before starting, ensure each VM is clean with no leftover Kubernetes config.

```bash
kubeadm reset -f

systemctl stop kubelet
systemctl stop crio 2>/dev/null || systemctl stop containerd

rm -rf /etc/kubernetes
rm -rf /var/lib/kubelet
rm -rf /etc/cni
rm -rf /opt/cni
rm -rf $HOME/.kube

iptables -F
iptables -t nat -F
iptables -t mangle -F
iptables -X

systemctl restart containerd
systemctl restart kubelet
```

---

## ⚙️ Phase 1 — OS Pre-configuration (All Nodes)

### Step 1 — Configure `/etc/hosts`

Add hostname entries to avoid DNS issues:

```bash
vi /etc/hosts
```

Append:

```
192.168.228.17   k8s-master
192.168.228.19   k8s-worker1
192.168.228.22   k8s-worker2
```

---

### Step 2 — Disable Swap ⚠️

Kubernetes requires swap to be disabled:

```bash
swapoff -a

# Make permanent
sed -i '/swap/d' /etc/fstab

# Verify (Swap line must show 0)
free -m
```

---

### Step 3 — Enable Required Kernel Modules

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

---

### Step 4 — Configure Sysctl for Networking

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply settings
sysctl --system
```

---

## 🐳 Phase 2 — Install Containerd (All Nodes)

### Option A — Install via DNF (Standard)

```bash
dnf install -y containerd
```

### Option B — Install via Docker Repo (if DNF fails)

> Use this if your OS is RHEL and shows subscription manager errors.

```bash
# Check OS
cat /etc/os-release

# Check enabled repos
subscription-manager repos --list-enabled

# Enable required repos (RHEL 8)
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms

# Add Docker repo as fallback
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install containerd
dnf install -y containerd.io
```

---

### Configure Containerd

```bash
# Generate default config
containerd config default | tee /etc/containerd/config.toml

# Enable SystemdCgroup (REQUIRED for Kubernetes)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Enable and restart
systemctl enable --now containerd
systemctl restart containerd

# Verify
systemctl status containerd   # Should show: active (running)
```

---

## 📦 Phase 3 — Install Kubernetes Components (All Nodes)

### Step 1 — Add Kubernetes Repository

```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
EOF
```

### Step 2 — Install kubeadm, kubelet, kubectl

```bash
dnf install -y kubelet kubeadm kubectl

systemctl enable --now kubelet
```

> ⚠️ `kubelet` will show a failure state until the cluster is initialized — this is **normal**.

---

## 🚀 Phase 4 — Initialize Cluster (Master Only)

### Step 1 — Run kubeadm init

```bash
kubeadm init --pod-network-cidr=192.168.0.0/16
```

> `192.168.0.0/16` is required for Calico CNI.

After success, copy the `kubeadm join` command shown at the end — you'll need it for worker nodes.

---

### Step 2 — Configure kubectl for Root User

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Test
kubectl get nodes
# Status will show NotReady (CNI not installed yet — this is normal)
```

---

## 🌐 Phase 5 — Install Calico CNI (Master Only)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait 1–2 minutes, then verify:

```bash
kubectl get pods -A
kubectl get nodes
# Master should now show: Ready
```

---

## 🔗 Phase 6 — Join Worker Nodes

Run the `kubeadm join` command (copied from Phase 4) on **both worker nodes**:

```bash
kubeadm join 192.168.228.17:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

> If you lost the join command, regenerate it on the master:
> ```bash
> kubeadm token create --print-join-command
> ```

---

## ✅ Phase 7 — Verify Cluster

Run on the **master node**:

```bash
kubectl get nodes
```

Expected output:

```
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   Xm    v1.29.x
k8s-worker1   Ready    <none>          Xm    v1.29.x
k8s-worker2   Ready    <none>          Xm    v1.29.x
```

```bash
kubectl get pods -A   # All pods should be Running or Completed
```

---

## 🛠️ Troubleshooting

| Issue | Fix |
|-------|-----|
| `kubelet` not starting | Check `journalctl -xeu kubelet` for errors |
| Nodes stuck at `NotReady` | Verify Calico pods are running: `kubectl get pods -n kube-system` |
| `kubeadm init` fails | Run Phase 0 cleanup and retry |
| containerd not found | Use Docker repo (Phase 2 Option B) |
| Swap not disabled | Re-run `swapoff -a` and verify with `free -m` |

---

## 📁 File Structure Reference

```
/etc/kubernetes/          ← Cluster config (master)
/var/lib/kubelet/         ← Kubelet data
/etc/containerd/          ← Containerd config
/etc/cni/                 ← CNI plugin config
~/.kube/config            ← kubectl credentials
/etc/yum.repos.d/         ← Package repos
```

---

## 📚 References

- [Kubernetes Official Docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Calico Docs](https://docs.tigera.io/calico/latest/getting-started/kubernetes/)
- [Containerd GitHub](https://github.com/containerd/containerd)

---

## 📝 License

MIT License — feel free to use and adapt this guide.
