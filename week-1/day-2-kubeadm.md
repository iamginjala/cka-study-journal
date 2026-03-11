# Day 2 — kubeadm Cluster Bootstrap

**Date:** 2026-03-10
**Week:** 1 | **Day:** 2
**Topics:** kubeadm init, kubeadm join, Static Pods, Certificates, kubeconfig, Killercoda hands-on

---

## 1. What is kubeadm?

kubeadm is a tool that **fast-tracks Kubernetes cluster creation**. Instead of manually configuring every component, kubeadm automates the entire setup.

### What kubeadm DOES:
- Generates all **TLS certificates** (`/etc/kubernetes/pki/`)
- Creates **kubeconfig files** for each component
- Deploys control plane components as **static pods**
- Installs kube-proxy and CoreDNS
- Prints the `kubeadm join` token for workers

### What kubeadm does NOT do:
- Does **not** install a CNI network plugin (you do this manually after init)
- Does **not** provide monitoring or observability
- Does **not** manage application deployments

> ⚠️ After `kubeadm init`, nodes stay `NotReady` until you install a CNI (Calico, Flannel, Weave, etc.)

---

## 2. The Two Key Commands

```
kubeadm init   →  sets up the CONTROL PLANE (run on master node)
kubeadm join   →  adds a WORKER NODE to the cluster
```

---

## 3. kubeadm init — What happens step by step

```
1. Checks prerequisites (CPU, memory, swap disabled, ports free)
        ↓
2. Generates all TLS certificates → /etc/kubernetes/pki/
        ↓
3. Creates kubeconfig files → /etc/kubernetes/*.conf
        ↓
4. Starts control plane as static pods → /etc/kubernetes/manifests/
   (etcd, kube-apiserver, kube-scheduler, kube-controller-manager)
        ↓
5. Installs kube-proxy and CoreDNS as regular pods
        ↓
6. Prints kubeadm join command with token for workers
```

### After kubeadm init — set up kubectl:
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Then install a CNI (example: Weave):
```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

---

## 4. kubeadm join — Joining a Worker Node

After `kubeadm init`, the output includes a join command like:

```bash
kubeadm join 172.30.1.2:6443 \
  --token y1pmcl.85cws5d22umykxd8 \
  --discovery-token-ca-cert-hash sha256:9a9e150652daf2c64e5ce7761d2a017df97e7e7c136c8e9bee29cad0ad6a0cb5
```

### Breaking it down:

| Part | What it means |
|---|---|
| `172.30.1.2:6443` | IP and port of the API server on control plane |
| `--token` | Temporary password proving this node is allowed to join. Expires in **24 hours** |
| `--discovery-token-ca-cert-hash` | Fingerprint of cluster CA — ensures node joins the RIGHT cluster |

### What happens during join:
```
node01 runs kubeadm join
    ↓
Contacts API server using the token
    ↓
API server verifies token ✅
    ↓
node01 downloads cluster info + certificates
    ↓
kubelet starts on node01
    ↓
node01 registers with API server
    ↓
Controller Manager sees new node → marks it Ready
```

---

## 5. Token Expired? One Command Fix

Token expires after 24 hours. If a worker needs to join after expiry:

```bash
# Run on CONTROL PLANE — creates new token + prints full join command
kubeadm token create --print-join-command
```

> 💡 This is the most commonly tested kubeadm scenario in CKA.

---

## 6. Static Pods — Critical Concept

Control plane components run as **static pods**, not regular pods.

| | Regular Pod | Static Pod |
|---|---|---|
| Created by | API server | kubelet directly |
| Stored in | etcd | `/etc/kubernetes/manifests/` |
| If deleted | Stays deleted | Recreated automatically |

### Static pod manifests location:
```bash
ls /etc/kubernetes/manifests/
# etcd.yaml
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml
```

> Editing these files → component restarts automatically. Used in etcd restore (Day 1) and cluster upgrades (Week 3).

---

## 7. Certificates — /etc/kubernetes/pki/

```
/etc/kubernetes/pki/
├── ca.crt                  ← cluster CA certificate
├── ca.key                  ← cluster CA private key
├── apiserver.crt           ← API server certificate
├── apiserver.key           ← API server private key
└── etcd/
    ├── ca.crt              ← etcd CA (--cacert in etcd backup)
    ├── server.crt          ← etcd server cert (--cert)
    └── server.key          ← etcd server key (--key)
```

---

## 8. kubeconfig Files — /etc/kubernetes/

Each component has its own identity (kubeconfig) to authenticate with the API server:

| File | Used by |
|---|---|
| `admin.conf` | kubectl (cluster admin) → copied to `~/.kube/config` |
| `super-admin.conf` | Emergency admin access |
| `kubelet.conf` | kubelet authentication to API server |
| `controller-manager.conf` | Controller manager authentication |
| `scheduler.conf` | Scheduler authentication |

> Think of each `.conf` as an **ID badge** — every component needs one to talk to the API server.

---

## 9. kubelet — Why it's Different

kubelet runs as a **systemd service**, not a pod.

```
Linux boots
    ↓
systemd starts (PID 1)
    ↓
systemd starts kubelet        ← systemd service
    ↓
kubelet reads /etc/kubernetes/manifests/
    ↓
kubelet starts etcd, API server, scheduler, controller-manager as static pods
```

> kubelet MUST start first — it's the one that launches everything else. It can't be a pod because nothing exists yet to run it.

### Check kubelet status:
```bash
systemctl status kubelet
systemctl restart kubelet   # if it's stopped
```

> ⚠️ kubelet runs on EVERY node — both control plane AND workers. If kubelet stops on any node → that node goes NotReady.

---

## 10. What runs on the Control Plane Node

```bash
crictl ps   # list all containers inside the node
```

| Container | Type | Started by |
|---|---|---|
| etcd | Static Pod | kubelet |
| kube-apiserver | Static Pod | kubelet |
| kube-scheduler | Static Pod | kubelet |
| kube-controller-manager | Static Pod | kubelet |
| coredns | Regular Pod | kubeadm |
| kube-proxy | Regular Pod | kubeadm |
| kindnet/CNI | Regular Pod | CNI installer |

---

## 11. Real Troubleshooting — Node NotReady After Restart

**Symptoms:**
```
kubectl get nodes → control-plane NotReady
kubectl describe node <name> → "Kubelet stopped posting node status"
```

**Root cause found today:**
KIND on Windows/WSL reassigns Docker container IPs after restart. kubelet was configured to reach API server at `172.20.0.2` but the container moved to `172.20.0.4`.

**Fix for KIND on Windows:**
```bash
kind delete cluster --name cka
kind create cluster --name cka --config ~/kind-cka.yaml
```

**Debugging steps used:**
```bash
kubectl describe node cka-control-plane | grep -A5 "Conditions:"
docker exec -it cka-control-plane bash
crictl ps | grep apiserver
curl -k https://172.20.0.2:6443/healthz
ip addr show | grep 172
```

---

## 12. Scenario Practice

### Scenario — Token expired, worker can't join
> Node `node02` needs to join the cluster but the token expired.

**Answer:** Run on control plane:
```bash
kubeadm token create --print-join-command
```
Copy output → run on node02 → verify with `kubectl get nodes`

---

## 13. Exam Tips

- After `kubeadm init` → always copy kubeconfig AND install CNI before nodes go Ready
- Token expires in 24h → `kubeadm token create --print-join-command` regenerates it
- Static pod won't start? → check `/etc/kubernetes/manifests/<component>.yaml` for errors
- Node NotReady? → `kubectl describe node` → Events + Conditions sections
- Always check context before starting exam question: `kubectl config current-context`

---

*Next: Day 3 — etcd Backup & Restore (hands-on drill)*
