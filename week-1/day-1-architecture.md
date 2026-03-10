# Day 1 — K8s Architecture & etcd Backup/Restore

**Date:** 2026-03-09
**Week:** 1 | **Day:** 1
**Topics:** Kubernetes Architecture Deep Dive + etcd Backup & Restore

---

## 1. What is Kubernetes?

Kubernetes (K8s) is a **container orchestration platform**.

### Why do we need it?

Containers alone have several limitations:
- They are **ephemeral** — they can die at any time
- No built-in **auto scaling**
- No built-in **load balancing**
- No **network communication** between containers by default (must be done manually)
- No **enterprise-level support**

Kubernetes solves all of the above.

---

## 2. K8s Architecture

Kubernetes has two main planes:

```
┌─────────────────────────────────────────────┐
│           CONTROL PLANE (Master)            │
│  ┌─────────────┐      ┌──────────────────┐  │
│  │  API Server │◄────►│      etcd        │  │
│  │   (heart)   │      │  (key-value DB)  │  │
│  └──────┬──────┘      └──────────────────┘  │
│         │                                   │
│  ┌──────▼──────┐      ┌──────────────────┐  │
│  │  Scheduler  │      │Controller Manager│  │
│  └─────────────┘      └──────────────────┘  │
└─────────────────────────────────────────────┘
                    │
        ┌───────────▼───────────┐
        │      WORKER NODE      │
        │  ┌────────────────┐   │
        │  │    kubelet     │   │
        │  ├────────────────┤   │
        │  │   kube-proxy   │   │
        │  ├────────────────┤   │
        │  │  Container     │   │
        │  │  Runtime       │   │
        │  │ (containerd)   │   │
        │  └────────────────┘   │
        │  [ Pod ][ Pod ][ Pod ] │
        └───────────────────────┘
```

---

## 3. Control Plane Components

### API Server — *"The Heart"*
- The **only entry point** into the cluster
- Exposes K8s to the external world (kubectl talks to this)
- The **only component that talks to etcd** directly
- Every other component communicates through the API server

### etcd — *"The Memory"*
- A **key-value database** that stores ALL cluster state
- Think of it as the cluster's metadata store
- If etcd goes down, the cluster loses its state
- **Must be backed up regularly** (see Section 5)

### Scheduler — *"The Placement Engine"*
- Watches for pods with no assigned node (`nodeName` is empty)
- Scores every node based on: CPU, memory, taints, affinity rules
- **Decides WHERE** a pod goes — but does NOT place it
- kubelet actually places it

> ⚠️ Scheduler only decides — kubelet executes.

### Controller Manager — *"The Watchdog"*
- Continuously watches **current state vs desired state**
- Takes corrective action when they differ
- Runs multiple controllers inside one process:

| Controller | Responsibility |
|---|---|
| ReplicaSet Controller | Ensures correct number of pod replicas |
| Node Controller | Monitors node health |
| Job Controller | Manages batch job pods |
| Deployment Controller | Manages rollouts/rollbacks |

> 🧠 Memory hook: **"RNJD"** — Replication, Node, Job, Deployment

---

## 4. Worker Node Components

### kubelet
- Agent running on every worker node
- Receives pod specs from the API server
- Talks to the container runtime (containerd) to start/stop containers
- Continuously reports **node and pod health** back to the API server

### kube-proxy
- Manages **iptables/ipvs rules** on each node
- Ensures Service IPs route to the correct pods
- Think of it as the **traffic cop** for pod networking

### Container Runtime
- The engine that actually runs containers (e.g. containerd, CRI-O)
- kubelet talks to it via CRI (Container Runtime Interface)

---

## 5. Communication Flow

> What happens when you run `kubectl apply -f deployment.yaml`?

```
kubectl apply
    ↓
API Server  ──► stores in etcd
    ↓
Controller Manager notices "desired=3 pods, actual=0" ──► creates 3 pod objects
    ↓
Scheduler sees 3 unscheduled pods ──► assigns a node to each
    ↓
kubelet on assigned node ──► reads pod spec from API server
    ↓
Container Runtime (containerd) ──► starts the container
```

---

## 6. Scenario Practice

### Scenario 1 — Pod stuck in Pending
> Deployment has 3 replicas but 1 pod is stuck in `Pending`.

**Answers:**
1. **Scheduler** decides pod placement
2. Pod is `Pending` because: insufficient CPU/memory, taint with no toleration, NodeAffinity mismatch, or no nodes available
3. **Controller Manager** (ReplicaSet controller) keeps retrying — it never gives up as long as the Deployment exists

**Debug command:**
```bash
kubectl describe pod <pending-pod-name>
# Look at the Events section at the bottom
```

### Scenario 2 — Team member deletes pods manually
> `webapp` deployment has 3 replicas. Team member runs `kubectl delete pod` on 2 of them.

**Answer:**
1. **Controller Manager** (ReplicaSet controller) notices current=1, desired=3
2. It immediately creates 2 new pods to restore desired state
3. If the **Deployment** itself is deleted → ReplicaSet is deleted → pods are deleted and **do NOT come back** because there is no longer anything watching for them

> 💡 Key insight: Pod → owned by ReplicaSet → owned by Deployment. Delete the owner and all children go with it.

---

## 7. etcd Backup & Restore

### Why back up etcd?
etcd holds **all cluster state** — every pod, deployment, secret, configmap, service. If it's lost, the cluster has no memory.

### Backup Command

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Breaking it down:**

| Part | What it does |
|---|---|
| `ETCDCTL_API=3` | Use etcd v3 API (always required for CKA) |
| `snapshot save <path>` | Save backup to this path (exam will specify it) |
| `--endpoints` | Where etcd is running — always `127.0.0.1:2379` on control plane |
| `--cacert` | CA certificate to trust |
| `--cert` | Client certificate (your identity) |
| `--key` | Private key |

> 📁 Path is YOUR choice (or the exam's). Use whatever path is specified in the question.

### Verify Backup

```bash
ETCDCTL_API=3 etcdctl snapshot status /opt/backup/etcd-backup.db
```

### Restore Command

```bash
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored
```

> ✅ Restore does NOT need certs — you're reading a local file, not talking to a live etcd.

### After Restore — Critical Step Most People Miss

Update the etcd static pod manifest to point to the new data directory:

```bash
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Find the line: --data-dir=/var/lib/etcd
# Change it to:  --data-dir=/var/lib/etcd-restored
```

etcd will automatically restart and pick up the restored data.

### Memory Hook for the 3 Certs

> **"CA, CRT, KEY"** — cacert → cert → key. All live under `/etc/kubernetes/pki/etcd/`

If you forget the cert paths during the exam:
```bash
kubectl describe pod etcd-controlplane -n kube-system | grep -E "cert|key|trusted"
```

---

## 8. Quick Reference

### Key Ports to Remember

| Component | Port |
|---|---|
| API Server | 6443 |
| etcd | 2379 (client), 2380 (peer) |
| kubelet | 10250 |
| kube-scheduler | 10259 |
| controller-manager | 10257 |

### Exam Tips
- `kubectl describe pod <name>` → Events section is always your first debug move
- For `Pending` pods: scheduler is the blocker, check resources/taints/affinity
- etcd backup: exam will give you the exact save path — use it exactly
- After etcd restore: always update `--data-dir` in `/etc/kubernetes/manifests/etcd.yaml`

---

*Next: Day 2 — kubeadm Cluster Bootstrap*