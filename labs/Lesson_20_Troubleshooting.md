# Lesson 20 — Logging, Monitoring & Troubleshooting — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts

- **`kubectl logs`** — stream container stdout/stderr. Add `-p`/`--previous`
  for the last-crashed container.
- **`kubectl describe`** — events, conditions, mounts, status — your first
  stop on any "why isn't this working".
- **`kubectl get events`** — namespace-wide event history (sort by `lastTimestamp`).
- **`kubectl top`** — live CPU/memory (needs metrics-server).
- **`kubectl debug`** — attach an ephemeral debug container to a running Pod.
- **Pod statuses**: `Pending`, `Running`, `Succeeded`, `Failed`,
  `Unknown`. Container reasons you'll see: `ImagePullBackOff`,
  `CrashLoopBackOff`, `OOMKilled`, `ContainerCreating`.

---

## Examples

### 1. `top` — node & pod resource consumption

```bash
kubectl top nodes
kubectl top pod -n kube-system --sort-by=cpu | head -10
```

Sample:

```
NAME                    CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
cka-lab-control-plane   78m          0%       774Mi           5%
cka-lab-worker          14m          0%       233Mi           1%
cka-lab-worker2         11m          0%       194Mi           1%
```

`kubectl top pod` may say `metrics not available yet` for the first ~30s after
a Pod starts — that's normal.

### 2. Logs

```bash
POD=$(kubectl -n l11 get pod -l app=busy -o name | head -1)

kubectl -n l11 logs $POD --tail=20
kubectl -n l11 logs $POD --since=2m
kubectl -n l11 logs $POD --previous          # last-restarted container
kubectl -n l11 logs $POD -f                  # follow
kubectl -n l11 logs -l app=busy --all-containers --max-log-requests=10
```

### 3. Events (sorted)

```bash
kubectl -n l11 get events --sort-by=.lastTimestamp | tail
```

Events live ~1h by default — investigate quickly.

### 4. Describe a Pod (look at the `Events:` section at the bottom)

```bash
kubectl -n l11 describe pod $POD | sed -n '/^Events:/,$p'
```

### 5. Diagnose `ImagePullBackOff`

```yaml
apiVersion: v1
kind: Pod
metadata: {name: missing-image, namespace: l11}
spec:
  containers:
  - name: app
    image: nginx:9.99-does-not-exist
```

```bash
kubectl -n l11 get pod missing-image
# missing-image   0/1   ErrImagePull   0   8s

kubectl -n l11 describe pod missing-image | grep -E 'ImagePull|Failed' | head -3
# Warning Failed ... Failed to pull image "nginx:9.99-does-not-exist": ... not found
```

### 6. Diagnose a crashing app

```yaml
apiVersion: v1
kind: Pod
metadata: {name: crashy, namespace: l11}
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","echo starting; sleep 2; echo failing; exit 1"]
```

```bash
kubectl -n l11 logs crashy
# starting
# failing
kubectl -n l11 get pod crashy
# crashy   0/1   Error   0   5s
```

If the Pod is in `CrashLoopBackOff` and the latest container has been replaced,
look at the previous logs:

```bash
kubectl -n l11 logs crashy --previous
```

### 7. Exec into a Pod / open a shell

```bash
kubectl -n l11 exec -it $POD -- sh
# Inside: ls /proc/1/, env, ps -ef, curl http://service/, etc.
```

### 8. `kubectl debug` — attach an ephemeral container

For distroless / minimal images where you can't shell in:

```bash
kubectl -n l11 debug $POD -it --image=busybox:1.37 \
  --target=app --share-processes -- sh
```

Inside the ephemeral container you can read files on the target container's
filesystem (`/proc/1/root/...`) and inspect its processes — without modifying
the running Pod.

### 9. Node troubleshooting

```bash
kubectl get nodes
kubectl describe node cka-lab-worker | sed -n '/Conditions:/,/Capacity:/p'
# Look for: Ready=True, MemoryPressure=False, DiskPressure=False, PIDPressure=False
```

On a real cluster, also check the host:

```bash
ssh worker
journalctl -u kubelet --since "10 min ago"
journalctl -u containerd --since "10 min ago"
systemctl status kubelet
```

---

## Lab

**Goal**: triage a small "outage": a Deployment shipped with three broken
configurations. Find each problem from `kubectl` output alone.

### Setup (run on a fresh namespace `l11-lab`)

```bash
kubectl create ns l11-lab

# A — missing image
kubectl -n l11-lab create deployment broken-image --image=nginx:9.99-nope

# B — crashloop (the command exits)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: crashloop, namespace: l11-lab}
spec:
  replicas: 1
  selector: {matchLabels: {app: crashloop}}
  template:
    metadata: {labels: {app: crashloop}}
    spec:
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh","-c","echo starting; exit 1"]
EOF

# C — Pending due to impossible nodeSelector
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: unschedulable, namespace: l11-lab}
spec:
  replicas: 1
  selector: {matchLabels: {app: unsched}}
  template:
    metadata: {labels: {app: unsched}}
    spec:
      nodeSelector: {disktype: ssd}     # no node has this label
      containers: [{name: app, image: nginx:1.27}]
EOF
```

### Steps

1. List all Pods in `l11-lab` and note their statuses.
2. For each one, identify the root cause using **only** `kubectl`. Write down
   the command you used.
3. Fix each Deployment with the smallest possible change so all Pods become
   `Running`.

### Solution

```bash
# 1
kubectl -n l11-lab get pod

# 2
kubectl -n l11-lab describe pod -l app=broken-image | sed -n '/^Events:/,$p'
# → "Failed to pull image nginx:9.99-nope ... not found"

kubectl -n l11-lab logs -l app=crashloop --tail=20
# → "starting" (then exits, kubelet restarts → CrashLoopBackOff)

kubectl -n l11-lab describe pod -l app=unsched | grep -A3 'Events:'
# → "0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector"

# 3
# A — patch to a real image
kubectl -n l11-lab set image deployment/broken-image nginx=nginx:1.27

# B — replace the failing command with something that stays up
kubectl -n l11-lab patch deployment crashloop --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/command","value":["sh","-c","echo healthy; sleep infinity"]}
]'

# C — drop the unsatisfiable nodeSelector
kubectl -n l11-lab patch deployment unschedulable --type=json -p='[
  {"op":"remove","path":"/spec/template/spec/nodeSelector"}
]'

# Wait
kubectl -n l11-lab rollout status deployment/broken-image
kubectl -n l11-lab rollout status deployment/crashloop
kubectl -n l11-lab rollout status deployment/unschedulable
```

### Verification

```bash
kubectl -n l11-lab get pod
# All Pods 1/1 Running, 0 restarts (or stable restart count).
```

### Cleanup

```bash
kubectl delete ns l11-lab
```
