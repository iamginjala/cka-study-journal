# Day 5 — Deployments & ReplicaSets

**Date:** 2026-03-13
**Week:** 1 | **Day:** 5
**Topics:** ReplicaSet, Deployment lifecycle, rollout, rollback, scaling, troubleshooting

---

## 1. ReplicaSet

A ReplicaSet ensures a specified number of pod replicas are running at all times.

### Create a ReplicaSet:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

### Scale ReplicaSet:
```bash
# From command line
kubectl scale replicaset/nginx-rs --replicas=6

# From YAML - edit replicas field and apply
kubectl apply -f replicaset.yaml
```

> 💡 In practice, you rarely create ReplicaSets directly — Deployments manage them for you.

---

## 2. Deployment

A Deployment manages ReplicaSets and provides declarative updates, rollouts, and rollbacks.

### Ownership chain:
```
Deployment → owns → ReplicaSet → owns → Pods
```

### Create a Deployment:
```bash
# Imperative
kubectl create deployment nginx --image=nginx:1.23.0 --replicas=3

# With labels
kubectl create deployment nginx --image=nginx:1.23.0 --replicas=3 $do > deploy.yaml
# Then add labels manually and apply
```

### Declarative:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: v1
  template:
    metadata:
      labels:
        app: v1
    spec:
      containers:
      - name: nginx
        image: nginx:1.23.0
```

---

## 3. Core Deployment Commands

```bash
# Create
kubectl create deployment nginx --image=nginx:1.23.0 --replicas=3

# List deployments
kubectl get deployments
kubectl get deploy

# Describe
kubectl describe deployment nginx

# Update image (fastest way)
kubectl set image deployment/nginx nginx=nginx:1.23.4

# Scale
kubectl scale deployment/nginx --replicas=5

# Delete
kubectl delete deployment nginx
```

---

## 4. Rollout Commands

```bash
# Check rollout status
kubectl rollout status deployment/nginx

# View rollout history
kubectl rollout history deployment/nginx

# Add change cause to current revision
kubectl annotate deployment/nginx kubernetes.io/change-cause="Pick up patch version"

# Rollback to previous revision
kubectl rollout undo deployment/nginx

# Rollback to specific revision
kubectl rollout undo deployment/nginx --to-revision=1
```

### View specific revision details:
```bash
kubectl rollout history deployment/nginx --revision=1
```

---

## 5. Updating a Deployment

### Method 1 — kubectl set image (fastest, exam recommended):
```bash
kubectl set image deployment/nginx nginx=nginx:1.23.4
```

### Method 2 — kubectl edit (slower, opens editor):
```bash
kubectl edit deployment/nginx
# Find image line, update manually, save
```

### Method 3 — Edit YAML and apply:
```bash
# Edit deploy.yaml → change image
kubectl apply -f deploy.yaml
```

> ⚠️ In CKA exam always use `kubectl set image` — it's one line and done.

---

## 6. Verify Rollout

```bash
# Check all pods are using new image
kubectl describe deployment nginx | grep Image
kubectl get pods -o wide
kubectl rollout status deployment/nginx   # shows "successfully rolled out"
```

---

## 7. Common YAML Errors & Fixes

### Error 1 — Wrong apiVersion
```yaml
# Wrong
apiVersion: v1
kind: Deployment

# Correct
apiVersion: apps/v1
kind: Deployment
```

**apiVersion reference:**

| Resource | apiVersion |
|---|---|
| Pod | `v1` |
| Service | `v1` |
| ConfigMap | `v1` |
| Secret | `v1` |
| Deployment | `apps/v1` |
| ReplicaSet | `apps/v1` |
| DaemonSet | `apps/v1` |
| StatefulSet | `apps/v1` |

> If you forget: `kubectl explain deployment | head -5`

---

### Error 2 — selector doesn't match pod template labels

```yaml
# Wrong - selector and template labels don't match
template:
  metadata:
    labels:
      env: demo      # pod label is "demo"
selector:
  matchLabels:
    env: dev         # ❌ selector looks for "dev" → no pods found

# Correct
template:
  metadata:
    labels:
      env: demo
selector:
  matchLabels:
    env: demo        # ✅ matches pod label
```

> 💡 Golden rule: `selector.matchLabels` MUST always match `template.metadata.labels`. If they don't match, the Deployment can't find or manage its own pods.

---

## 8. Command Corrections (Common Mistakes)

| Wrong | Correct |
|---|---|
| `kubectl rollback undo` | `kubectl rollout undo` |
| `kubectl --scale replicas=5` | `kubectl scale --replicas=5` |
| `kubectl edit` for image update | `kubectl set image` (faster) |

---

## 9. Full Deployment Workflow — Exam Scenario

```bash
# 1. Create deployment
kubectl create deployment nginx --image=nginx:1.23.0 --replicas=3

# 2. Verify
kubectl get deployment nginx
kubectl get pods

# 3. Update image
kubectl set image deployment/nginx nginx=nginx:1.23.4

# 4. Add change cause
kubectl annotate deployment/nginx kubernetes.io/change-cause="Pick up patch version"

# 5. Scale
kubectl scale deployment/nginx --replicas=5

# 6. View history
kubectl rollout history deployment/nginx

# 7. Rollback to revision 1
kubectl rollout undo deployment/nginx --to-revision=1

# 8. Verify image rolled back
kubectl describe deployment nginx | grep Image
```

---

## 10. Exam Tips

- Always use `apps/v1` for Deployment, ReplicaSet, DaemonSet, StatefulSet
- `selector.matchLabels` must match `template.metadata.labels` exactly
- `kubectl set image` is faster than `kubectl edit` for image updates
- Always add change cause with `kubectl annotate` after updating — exam often asks for it
- `kubectl rollout undo --to-revision=1` to go to specific revision
- Verify rollback with `kubectl describe deployment | grep Image`

---

*Next: Day 6 — Lab Day (Cluster Admin practice)*
