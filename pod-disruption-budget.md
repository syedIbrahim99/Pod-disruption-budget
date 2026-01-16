# Pod Disruption Budget (PDB) – Drain Behavior Explained (Hands‑On)

---

## 1. Cluster Setup (Example)

We have a Kubernetes cluster with **1 master and 3 worker nodes**.

```
master-1

worker-1 → nginx-pod-1, nginx-pod-2
worker-2 → nginx-pod-3, nginx-pod-4
worker-3 → nginx-pod-5, nginx-pod-6
```

### Assumptions

* `nginx` is managed by a **Deployment**
* Replica count = **6**
* Pods are evenly distributed by the scheduler

---

## 2. NGINX Deployment (YAML)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 6
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

📌 This deployment ensures **6 nginx pods** are always desired.

---

## 3. PodDisruptionBudget Configuration (YAML)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 4
  selector:
    matchLabels:
      app: nginx
```

### Meaning

* At least **4 nginx pods must be AVAILABLE at all times**
* AVAILABLE = `Running + Ready`

---

## 4. Very Important Concepts (Must Understand)

### What PDB Checks

PDB checks **only one thing** during eviction:

> If this pod is evicted now, will AVAILABLE pods fall below `minAvailable`?

### What PDB Does NOT Check

* Node capacity
* Whether other nodes have space
* Whether new pods can be scheduled later
* Whether new nodes can be added

📌 **PDB is enforced at eviction time, not scheduling time**

---

## 5. Hands‑On Walkthrough (Step by Step)

### Step 1: Deploy NGINX

```bash
kubectl apply -f nginx-deployment.yaml
```

Verify:

```bash
kubectl get pods -o wide
```

You should see 6 pods distributed across worker nodes.

---

### Step 2: Apply PodDisruptionBudget

```bash
kubectl apply -f nginx-pdb.yaml
```

Verify:

```bash
kubectl get pdb
```

Expected output:

```
NAME        MIN AVAILABLE   ALLOWED DISRUPTIONS
nginx-pdb  4               2
```

---

## 6. Drain Worker‑3 (Allowed Case)

### Pods on worker‑3

```
nginx-pod-5
nginx-pod-6
```

### Action

```bash
kubectl drain worker-3 --ignore-daemonsets
```

### Availability Calculation

| State                 | Available Pods |
| --------------------- | -------------- |
| Before eviction       | 6              |
| After evicting 2 pods | 4              |

✅ PDB condition satisfied (`4 ≥ minAvailable 4`)

### Result

Pods are recreated on other nodes:

```
worker-1 → nginx-pod-1, nginx-pod-2, nginx-pod-5
worker-2 → nginx-pod-3, nginx-pod-4, nginx-pod-6
worker-3 → drained
```

Drain **succeeds**.

---

## 7. Drain Worker‑2 (Blocked Case)

### Pods on worker‑2

```
nginx-pod-3
nginx-pod-4
nginx-pod-6
```

### Drain Command

```bash
kubectl drain worker-2 --ignore-daemonsets
```

### Eviction Happens One Pod at a Time

| Eviction Step | Available Pods | Allowed              |
| ------------- | -------------- | -------------------- |
| Evict pod 1   | 6 → 5          | ✅ Yes                |
| Evict pod 2   | 5 → 4          | ✅ Yes                |
| Evict pod 3   | 4 → 3          | ❌ No (PDB violation) |

🚨 Drain stops immediately.

---

## 8. Common Misunderstanding (Very Important)

❌ Wrong assumption:

> Drain failed because pods cannot be scheduled on other nodes

✅ Correct reason:

> Drain failed because evicting another pod would reduce AVAILABLE pods below `minAvailable`

📌 Scheduler is **not involved yet**.

---

## 9. Maximum Allowed Disruption

```
Total pods = 6
minAvailable = 4

Allowed disruptions = 6 - 4 = 2
```

➡️ Only **2 nginx pods** can be disrupted at a time.

---

## 10. How to Successfully Drain Worker‑2

### Option 1: Scale Deployment (YAML)

```yaml
spec:
  replicas: 7
```

Then:

```
Allowed disruptions = 7 - 4 = 3
```

---

### Option 2: Relax the PDB

```yaml
spec:
  minAvailable: 3
```

---

### Option 3: Add a New Worker Node

⚠️ Important:

> Pods must already be **Running & Ready** on other nodes **before eviction starts**

---

## 11. Key Takeaway

**PodDisruptionBudget protects application availability, not node maintenance convenience.**

Eviction safety is checked first. Scheduling happens later.
