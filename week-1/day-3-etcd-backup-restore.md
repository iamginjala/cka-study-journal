# Day 3 — etcd Backup & Restore

**Date:** 2026-03-11
**Week:** 1 | **Day:** 3
**Topics:** etcd backup methods, snapshot save, snapshot restore, hands-on on Killercoda

---

## 1. Why Back Up etcd?

etcd holds **all cluster state** — every pod, deployment, secret, configmap, service. If etcd is lost or corrupted, the cluster has no memory. Regular backups are critical for disaster recovery.

---

## 2. Two Backup Methods

### Method 1 — Built-in Snapshot (most common, CKA tested)
Talks to the **live etcd process** over the network → needs TLS credentials.

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-backup.db \
  --endpoints=127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Method 2 — Volume Snapshot
Filesystem-level copy of the etcd data directory. No network connection → no certs needed. Done at the infrastructure level (cloud snapshots, LVM snapshots).

---

## 3. How to Find etcd Endpoint and Cert Paths

Never guess — always verify from the etcd manifest:

```bash
# Find endpoint
cat /etc/kubernetes/manifests/etcd.yaml | grep listen-client

# Find cert paths
cat /etc/kubernetes/manifests/etcd.yaml | grep -E "cert|key|trusted"
```

**Default values (memorize these):**

| Parameter | Default Value |
|---|---|
| `--endpoints` | `127.0.0.1:2379` |
| `--cacert` | `/etc/kubernetes/pki/etcd/ca.crt` |
| `--cert` | `/etc/kubernetes/pki/etcd/server.crt` |
| `--key` | `/etc/kubernetes/pki/etcd/server.key` |

> ⚠️ etcd uses its OWN CA under `/etc/kubernetes/pki/etcd/ca.crt` — NOT the cluster CA at `/etc/kubernetes/pki/ca.crt`

---

## 4. Verify the Backup

```bash
etcdutl --write-out=table snapshot status /backup/etcd-backup.db
```

Shows hash, revision, total keys, and size — confirms backup is valid.

---

## 5. Restore — Full Step by Step

### Step 1 — Stop etcd and API server (move static pod manifests out)

```bash
mkdir -p /tmp/manifests-backup
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
```

> kubelet sees manifests gone → automatically stops both pods.
> ⚠️ Wait ~30 seconds for pods to fully stop before restoring.

### Step 2 — Restore the snapshot

```bash
# Use etcdutl NOT etcdctl for restore
etcdutl snapshot restore /backup/etcd-backup.db --data-dir /var/lib/etcd-restore
```

> ⚠️ `etcdctl snapshot restore` → throws "unknown flag" error
> ✅ Always use `etcdutl` for restore

### Step 3 — Update etcd manifest to use new data directory

```bash
vi /tmp/etcd.yaml
```

Find and change:
```yaml
# Before
--data-dir=/var/lib/etcd

# After
--data-dir=/var/lib/etcd-restore
```

Also update the volume mount if present:
```yaml
volumes:
- hostPath:
    path: /var/lib/etcd-restore   # update this too
    type: DirectoryOrCreate
  name: etcd-data
```

### Step 4 — Move manifests back

```bash
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

kubelet sees manifests return → automatically restarts both pods.

### Step 5 — Verify

```bash
# Wait ~30-60 seconds then:
kubectl get nodes
kubectl get pods -n kube-system | grep etcd
```

---

## 6. Complete Flow Diagram

```
BACKUP:
kubectl → etcdctl snapshot save → /backup/etcd-backup.db
                ↑
        needs TLS certs (cacert, cert, key)
        needs endpoint (127.0.0.1:2379)

RESTORE:
mv manifests to /tmp     → stops etcd + API server
        ↓
etcdutl snapshot restore → creates /var/lib/etcd-restore
        ↓
update etcd.yaml         → point --data-dir to new location
        ↓
mv manifests back        → kubelet restarts components
        ↓
kubectl get nodes        → verify cluster healthy
```

---

## 7. Common Mistakes & Fixes

| Mistake | Error | Fix |
|---|---|---|
| `ETCDCTL_API = 3` (spaces) | `ETCDCTL_API: command not found` | No spaces: `ETCDCTL_API=3` |
| `--cacert= /path` (space after =) | parse error | No space: `--cacert=/path` |
| `etcdctl snapshot restore` | `unknown flag: --data-dir` | Use `etcdutl` for restore |
| `/backup/` doesn't exist | `no such file or directory` | `mkdir -p /backup` first |
| Wrong CA path (`pki/ca.crt`) | auth error | Use `pki/etcd/ca.crt` |
| Typo `ectd` instead of `etcd` | file not found | Double check cert paths |

---

## 8. etcdctl vs etcdutl

| Tool | Use for |
|---|---|
| `etcdctl` | snapshot **save**, cluster operations |
| `etcdutl` | snapshot **restore**, snapshot **status** |

> This distinction is commonly tested — using the wrong tool gives a cryptic error.

---

## 9. Exam Tips

- Exam question will specify the exact backup path — use it exactly
- Always `mkdir -p <backup-dir>` before saving
- Always verify with `etcdutl snapshot status` after backup
- Stop API server AND etcd before restore — both need to be down
- After restore, update BOTH `--data-dir` flag AND volume hostPath in etcd.yaml
- `kubectl get nodes` after restore confirms cluster is back to healthy state
- If asked for endpoint during exam: `cat /etc/kubernetes/manifests/etcd.yaml | grep listen-client`

---

## 10. Quick Reference — Copy-Paste Commands

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-backup.db --endpoints=127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key

# Verify
etcdutl --write-out=table snapshot status /backup/etcd-backup.db

# Stop static pods
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore
etcdutl snapshot restore /backup/etcd-backup.db --data-dir /var/lib/etcd-restore

# Restart static pods
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Verify cluster
kubectl get nodes
```

---

*Next: Day 4 — kubectl Mastery & Imperative Commands*
