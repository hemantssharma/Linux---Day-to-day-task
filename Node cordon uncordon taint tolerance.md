```markdown
# Kubernetes Node Management & Scheduling – Complete Guide

This repository documents **core Kubernetes node operations and scheduling concepts**, along with **hands-on commands** using both **kubectl** and **k9s**.

It covers:
- Cordon / Uncordon
- Drain
- Taints & Tolerations
- PodDisruptionBudgets (PDBs)
- StatefulSet behavior during drain
- When to use each mechanism

---

## 📌 High-Level Mental Model

```

Cordon        → Stop new pods
Drain         → Move existing pods safely
Taints        → Repel pods
Tolerations   → Allow exceptions
PDB           → Safety guardrail
StatefulSet   → Ordered, data-safe movement

````

---

# SEGMENT 1: Kubernetes (kubectl) Commands & Concepts

---

## 1️⃣ Cordon a Node

### What it does
- Marks the node as **unschedulable**
- **Existing pods keep running**
- **No new pods** are scheduled

### Command
```bash
kubectl cordon <node-name>
````

### Use when

* Debugging a node
* Letting running jobs finish
* Preparing for maintenance

---

## 2️⃣ Uncordon a Node

### What it does

* Makes the node **schedulable again**

### Command

```bash
kubectl uncordon <node-name>
```

---

## 3️⃣ Drain a Node

### What it does

* Automatically **cordons** the node
* **Evicts existing workload pods**
* Pods managed by controllers are **recreated on other nodes**
* DaemonSets are **ignored**

### Command

```bash
kubectl drain <node-name> --ignore-daemonsets
```

### Common additional flags

```bash
--delete-emptydir-data   # Required if pods use emptyDir
--force                  # Needed for standalone pods
```

---

## 4️⃣ Cordon vs Drain (Important Difference)

| Action | New Pods | Existing Pods            |
| ------ | -------- | ------------------------ |
| Cordon | ❌ No     | ✅ Keep running           |
| Drain  | ❌ No     | 🔄 Evicted & rescheduled |

---

## 5️⃣ PodDisruptionBudgets (PDBs)

### What they do

* Limit how many pods can be unavailable at once
* Protect availability during voluntary disruptions (drain, rollout)

### Example

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

### Effect during drain

* If eviction violates the PDB → ❌ drain is blocked
* Prevents accidental downtime

---

## 6️⃣ StatefulSets During Drain

### What happens

* Pods are evicted **one at a time**
* Pod identity is preserved (`db-0`, `db-1`)
* PVCs are retained
* Ordering is respected

### Common drain failures

1. **PDB blocks eviction**
2. **Local PersistentVolumes** can’t move
3. **Single replica StatefulSet** causes downtime

### Fix strategies

* Temporarily relax PDB
* Scale replicas up
* Avoid local-only storage for movable workloads

---

## 7️⃣ Taints & Tolerations

### What problem they solve

Prevent random pods from running on special-purpose nodes.

---

### Taint format

```text
key=value:effect
```

Example:

```bash
kubectl taint node node1 role=db:NoSchedule
```

---

## 8️⃣ Taint Effects Explained

### 🔴 NoSchedule (hard rule)

* New pods are blocked
* Existing pods continue running

### 🟡 PreferNoSchedule (soft rule)

* Scheduler avoids the node if possible
* Pod may still land there

### 🔥 NoExecute (eviction rule)

* New pods blocked
* Existing pods **evicted unless tolerated**

---

## 9️⃣ Tolerations (Applied to Pods)

### Example toleration

```yaml
tolerations:
- key: "role"
  operator: "Equal"
  value: "db"
  effect: "NoSchedule"
```

### Operator types

* `Equal` → key + value must match
* `Exists` → value is ignored

---

### NoExecute with tolerationSeconds

```yaml
tolerations:
- key: "maintenance"
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 300
```

➡️ Pod is evicted after 5 minutes

---

## 10️⃣ Taints vs Labels (Very Important)

| Feature          | Taints          | Labels                  |
| ---------------- | --------------- | ----------------------- |
| Purpose          | Repel pods      | Attract pods            |
| Applied to       | Nodes           | Nodes                   |
| Used with        | Tolerations     | NodeSelector / Affinity |
| Eviction capable | Yes (NoExecute) | No                      |

---

# SEGMENT 2: k9s (Interactive UI) Operations

---

## 1️⃣ Open k9s

```bash
k9s
```

---

## 2️⃣ Navigate to Nodes View

```text
:nodes
```

or

```text
:no
```

---

## 3️⃣ Cordon a Node

* Select node
* Press:

```text
c
```

---

## 4️⃣ Uncordon a Node

* Select node
* Press:

```text
u
```

---

## 5️⃣ Drain a Node

* Select node
* Press:

```text
d
```

* Confirm the action

k9s handles:

* Ignoring DaemonSets
* Graceful eviction
* Error reporting if PDBs block drain

---

## 6️⃣ Action Menu (Alternative)

* Select node
* Press:

```text
a
```

* Choose:

  * Cordon
  * Drain
  * Uncordon

---

## 7️⃣ Viewing Drain Failures in k9s

k9s clearly shows:

* Which pod is blocking the drain
* Whether it’s due to:

  * PDB
  * emptyDir
  * Standalone pod

---

# When to Use What (Quick Guide)

| Scenario                 | Recommended Action |
| ------------------------ | ------------------ |
| Debugging node           | Cordon             |
| OS / kernel upgrade      | Cordon + Drain     |
| Scaling down nodes       | Cordon + Drain     |
| Dedicated workload nodes | Taints             |
| Emergency eviction       | NoExecute taint    |

---

# Final Takeaway 🧠

```
Cordon      → Stop new pods
Drain       → Move existing pods
Taints      → Node says "stay away"
Toleration  → Pod says "I’m allowed"
PDB         → Safety net
StatefulSet → Ordered, data-safe relocation
```

