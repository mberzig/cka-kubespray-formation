# Lesson 12 — Scheduling — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts

| Mechanism                  | Says                                                     |
|----------------------------|----------------------------------------------------------|
| `nodeSelector`             | "Schedule me only on a node with these labels."          |
| `taints` + `tolerations`   | Repel Pods from a node unless they explicitly tolerate. |
| `nodeAffinity`             | Richer node-targeting (operators In/NotIn/Exists…).      |
| `podAffinity` / `podAntiAffinity` | Co-locate or spread Pods based on other Pods.    |
| `topologySpreadConstraints` | Cluster-wide spreading (zones, nodes, racks).            |
| `resources.requests/limits`| Drive scheduling decisions and QoS class.                |

QoS classes:

- `Guaranteed` — all containers have `requests == limits` for cpu & memory.
- `Burstable` — at least one resource has `request < limit` (or only request).
- `BestEffort` — no requests, no limits.

---

## Examples

### 1. Label nodes for selection

```bash
kubectl label node cka-lab-worker  disktype=ssd
kubectl label node cka-lab-worker2 disktype=hdd
kubectl get nodes -L disktype
```

### 2. `nodeSelector`

```yaml
apiVersion: v1
kind: Pod
metadata: {name: ssd-only, namespace: l8}
spec:
  nodeSelector: {disktype: ssd}
  containers: [{name: app, image: nginx:1.27}]
```

```bash
kubectl -n l8 get pod ssd-only -o jsonpath='{.spec.nodeName}'
# cka-lab-worker
```

### 3. Taint + toleration

```bash
kubectl taint nodes cka-lab-worker2 dedicated=batch:NoSchedule
```

Pod **with** matching toleration → schedules on worker2; **without** → stays
`Pending`:

```yaml
# Pod that tolerates the taint
apiVersion: v1
kind: Pod
metadata: {name: tolerator, namespace: l8}
spec:
  tolerations:
  - {key: dedicated, operator: Equal, value: batch, effect: NoSchedule}
  nodeSelector: {disktype: hdd}
  containers: [{name: app, image: nginx:1.27}]
---
# Pod that does NOT tolerate the taint, but targets worker2 via nodeSelector
apiVersion: v1
kind: Pod
metadata: {name: no-tolerator, namespace: l8}
spec:
  nodeSelector: {disktype: hdd}
  containers: [{name: app, image: nginx:1.27}]
```

```bash
kubectl -n l8 get pod tolerator no-tolerator
# tolerator    Running   ...
# no-tolerator Pending   ...
```

Remove the taint:

```bash
kubectl taint nodes cka-lab-worker2 dedicated=batch:NoSchedule-
```

### 4. `nodeAffinity` (richer than `nodeSelector`)

```yaml
apiVersion: v1
kind: Pod
metadata: {name: aff-ssd, namespace: l8}
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: [ssd]
  containers: [{name: app, image: nginx:1.27}]
```

`required…` is a hard constraint. Soft preference is
`preferredDuringSchedulingIgnoredDuringExecution` (with a weight).

### 5. `podAntiAffinity` — spread replicas across nodes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: spread, namespace: l8}
spec:
  replicas: 2
  selector: {matchLabels: {app: spread}}
  template:
    metadata: {labels: {app: spread}}
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels: {app: spread}
            topologyKey: kubernetes.io/hostname
      containers: [{name: nginx, image: nginx:1.27}]
```

Validated: the 2 replicas landed on different worker nodes.

### 6. Resource requests/limits → QoS class

```yaml
apiVersion: v1
kind: Pod
metadata: {name: limited, namespace: l8}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","sleep infinity"]
    resources:
      requests: {cpu: "50m", memory: "32Mi"}
      limits:   {cpu: "100m", memory: "64Mi"}
```

```bash
kubectl -n l8 get pod limited -o jsonpath='{.status.qosClass}'
# Burstable
```

---

## Lab

**Goal**: Move `batch` jobs to a dedicated worker; ensure stateless `web`
replicas spread across remaining workers; demonstrate that a Pod with no
toleration cannot reach the dedicated worker.

### Steps

1. Create namespace `l8-lab`.
2. Label `cka-lab-worker` with `role=web`, `cka-lab-worker2` with `role=batch`.
3. Taint `cka-lab-worker2` with `workload=batch:NoSchedule`.
4. Deploy `batch-job`: 1 replica, image `busybox:1.37`, command `sleep 600`,
   with the right toleration + `nodeSelector` so it lands on
   `cka-lab-worker2`.
5. Deploy `web` (`nginx:1.27`, 2 replicas) with `nodeSelector: role=web` and
   `podAntiAffinity` so replicas spread (here only one matching node — observe
   one Pending).
6. Re-target `web` with `nodeSelector: role in (web, batch)` and add the
   `workload=batch:NoSchedule` toleration. Both replicas should now schedule
   on different nodes.
7. Show that without the toleration a third replica targeted to `batch` cannot
   schedule.

### Solution

```bash
# 1
kubectl create ns l8-lab

# 2
kubectl label node cka-lab-worker  role=web   --overwrite
kubectl label node cka-lab-worker2 role=batch --overwrite

# 3
kubectl taint nodes cka-lab-worker2 workload=batch:NoSchedule --overwrite

# 4
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: batch-job, namespace: l8-lab}
spec:
  replicas: 1
  selector: {matchLabels: {app: batch-job}}
  template:
    metadata: {labels: {app: batch-job}}
    spec:
      nodeSelector: {role: batch}
      tolerations:
      - {key: workload, operator: Equal, value: batch, effect: NoSchedule}
      containers:
      - name: job
        image: busybox:1.37
        command: ["sh","-c","sleep 600"]
EOF

# 5
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: web, namespace: l8-lab}
spec:
  replicas: 2
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      nodeSelector: {role: web}
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector: {matchLabels: {app: web}}
            topologyKey: kubernetes.io/hostname
      containers: [{name: nginx, image: nginx:1.27}]
EOF
# Observe: one of the web pods will be Pending — only one node has role=web.
kubectl -n l8-lab get pod -l app=web -o wide

# 6 — broaden the selector + tolerate the batch taint
# A rolling update would briefly try to surge a 3rd Pod which can't satisfy the
# 1-per-node anti-affinity. Easiest in the lab: delete and recreate.
kubectl -n l8-lab delete deployment web
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: web, namespace: l8-lab}
spec:
  replicas: 2
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      tolerations:
      - {key: workload, operator: Equal, value: batch, effect: NoSchedule}
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - {key: role, operator: In, values: [web, batch]}
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector: {matchLabels: {app: web}}
            topologyKey: kubernetes.io/hostname
      containers: [{name: nginx, image: nginx:1.27}]
EOF
kubectl -n l8-lab rollout status deployment/web

# 7
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: stowaway, namespace: l8-lab}
spec:
  nodeSelector: {role: batch}     # targets worker2, but no toleration
  containers: [{name: app, image: nginx:1.27}]
EOF
sleep 5
kubectl -n l8-lab get pod stowaway      # Pending
kubectl -n l8-lab describe pod stowaway | grep -A1 'Events:'
```

### Verification

```bash
kubectl -n l8-lab get pod -o wide
# - batch-job-*    → cka-lab-worker2
# - web-* (2)      → one on each worker
# - stowaway       → Pending, "1 node(s) had untolerated taint {workload: batch}"
```

### Cleanup

```bash
kubectl delete ns l8-lab
kubectl taint nodes cka-lab-worker2 workload=batch:NoSchedule-
kubectl label node cka-lab-worker  role-
kubectl label node cka-lab-worker2 role-
```
