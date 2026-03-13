# Day 4 — kubectl Mastery (Part 1)

**Date:** 2026-03-12
**Week:** 1 | **Day:** 4
**Topics:** Imperative vs Declarative, YAML errors, Pod creation

---

## 1. What is kubectl?

kubectl is a **command line tool** to interact with Kubernetes. It can manage any Kubernetes resource — not just pods:

```
pods, deployments, services, configmaps,
secrets, namespaces, nodes, ingress, PVs...
```

### Core operations:

| Command | What it does |
|---|---|
| `kubectl get` | List resources |
| `kubectl describe` | Detailed info about a resource |
| `kubectl apply` | Create or update from YAML |
| `kubectl delete` | Delete a resource |
| `kubectl exec` | Run command inside a pod |
| `kubectl logs` | View pod logs |
| `kubectl edit` | Edit resource live |
| `kubectl explain` | Docs for any resource field |

---

## 2. Two Ways to Use kubectl

### Imperative — type the full command
```bash
kubectl run nginx --image=nginx
kubectl create deployment demo --image=nginx --replicas=3
```
- Fast, no file needed
- Best for CKA exam speed

### Declarative — apply a YAML file
```bash
kubectl apply -f pod.yaml
```
- Repeatable and version controllable
- Best for complex configs

> 💡 **CKA exam strategy:** Use imperative commands to generate YAML quickly, then edit if needed. Never write YAML from scratch.

---

## 3. The dry-run Pattern — Most Important CKA Trick

Generate YAML without creating the resource:

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml
kubectl create deployment demo --image=nginx --dry-run=client -o yaml > deploy.yaml
```

Using the `$do` alias set up in Day 1:
```bash
kubectl run nginx --image=nginx $do
kubectl create deployment demo --image=nginx $do > deploy.yaml
```

> Always use dry-run to generate a base YAML → edit it → apply it. Saves huge time in the exam.

---

## 4. Creating Pods

### Imperative:
```bash
kubectl run redis --image=redis
kubectl run nginx --image=nginx --port=80
```

### Declarative (pod.yaml):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
```

---

## 5. Multiple Resources in One YAML File

Use `---` as a separator between resources:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
  labels:
    app: test
spec:
  containers:
  - image: redis
    name: redis
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: demo
spec:
  containers:
  - image: nginx
    name: nginx
```

> Without `---`, Kubernetes only reads the first definition and ignores everything after.

---

## 6. Common YAML Errors & Fixes

### Error 1 — Wrong casing for kind
```
no kind "pod" is registered for version "v1"
```

```yaml
# Wrong
kind: pod

# Correct - always PascalCase
kind: Pod
kind: Deployment
kind: Service
kind: ConfigMap
```

> `kind` field → always lowercase
> Resource type → always PascalCase

---

### Error 2 — Missing `---` separator
**Symptom:** Only one pod created when two are defined in the file.

**Fix:** Add `---` between each resource definition.

---

### Error 3 — Wrong indentation
```
unknown field "spec.containers[0].apiVersion"
```

**What it means:**
```
spec
  containers[0]   ← inside first container
    apiVersion    ← Kubernetes found apiVersion HERE (wrong!)
```

Kubernetes found `apiVersion` nested inside a container definition. It doesn't belong there.

**Rule:** `apiVersion` and `kind` are ALWAYS top-level fields — zero indentation:

```yaml
# Wrong - apiVersion indented under container
spec:
  containers:
  - image: redis
    name: redis
    apiVersion: v1    # ❌ inside container

# Correct - apiVersion at root level
apiVersion: v1        # ✅ zero indentation
kind: Pod
```

> YAML indentation determines what belongs to what. Always double-check alignment.

---

## 7. YAML Golden Rules

1. `apiVersion` and `kind` → always at **zero indentation** (root level)
2. Resource types → always **PascalCase** (`Pod`, `Deployment`, `Service`)
3. Multiple resources in one file → separated by **`---`**
4. Indentation → always **2 spaces**, never tabs
5. Field names → always **camelCase** (`containerPort`, not `container_port`)

---

## 8. Cluster Morning Startup (KIND on Windows/WSL)

```bash
# 1. Open Docker Desktop first (wait for green icon)
# 2. Open WSL Ubuntu terminal
# 3. Run:
kind get clusters
docker ps
kubectl get nodes

# If control-plane NotReady:
kind delete cluster --name cka
kind create cluster --name cka --config ~/kind-cka.yaml
```

### Smart startup script (`~/cka-start.sh`):
```bash
#!/bin/bash

echo "Checking KIND cluster..."

if ! kind get clusters | grep -q "cka"; then
  echo "Cluster not found. Creating..."
  kind create cluster --name cka --config ~/kind-cka.yaml
else
  STATUS=$(kubectl get node cka-control-plane --no-headers 2>/dev/null | awk '{print $2}')
  if [ "$STATUS" != "Ready" ]; then
    echo "Control-plane NotReady. Recreating..."
    kind delete cluster --name cka
    kind create cluster --name cka --config ~/kind-cka.yaml
  else
    echo "Cluster healthy!"
  fi
fi

kubectl get nodes
```

```bash
chmod +x ~/cka-start.sh
~/cka-start.sh
```

---

## 9. Coming Up — Day 4 Part 2

- `$do` dry-run pattern hands-on
- Imperative commands: expose, create configmap, create secret
- Debugging: logs, exec, describe, events
- Output formats: `-o wide`, `-o yaml`, `-o jsonpath`

---

*Next: Day 4 Part 2 — kubectl Mastery continued*
