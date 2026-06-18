# Lesson 1 — Kubernetes Architecture — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts (quick recap)

A Kubernetes cluster has two planes:

- **Control plane** — runs the cluster brain (etcd, kube-apiserver,
  kube-controller-manager, kube-scheduler). No user workloads.
- **Worker plane** — runs user Pods. Each worker runs the container runtime,
  kubelet (manages Pods) and kube-proxy (Service networking).

Every node also runs a CNI plugin (kindnet, Calico, Flannel…) to provide the Pod
network.

---

## Examples

### 1. Inspect cluster identity & version

```bash
kubectl cluster-info
kubectl version
```

Expected output (excerpt):

```
Kubernetes control plane is running at https://127.0.0.1:41539
CoreDNS is running at https://127.0.0.1:41539/api/v1/...
Server Version: v1.31.2
```

### 2. List nodes and their roles

```bash
kubectl get nodes -o wide
```

The `ROLES` column shows `control-plane` for the master and `<none>` for
workers. `INTERNAL-IP` and `CONTAINER-RUNTIME` give the node IP and runtime
(containerd in our case).

### 3. Identify the control-plane components

The 4 mandatory control-plane Pods (`etcd`, `kube-apiserver`,
`kube-controller-manager`, `kube-scheduler`) run as **static Pods** on the
control-plane node:

```bash
kubectl get pods -n kube-system --field-selector spec.nodeName=cka-lab-control-plane \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase'
```

### 4. See where each system Pod runs

```bash
kubectl get pods -n kube-system \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName' --no-headers \
  | sort -k2
```

You should observe:
- `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` →
  control-plane only.
- `kube-proxy`, `kindnet` (the CNI) → **every** node (DaemonSets).
- `coredns` → typically the control-plane (scheduled wherever).

### 5. View the API resources exposed

```bash
kubectl api-resources --verbs=list -o name | head -20
kubectl api-versions | head
```

`api-resources` lists every Kind the apiserver knows about; `api-versions` lists
the API groups & versions.

---

## Lab

**Goal**: Map the running cluster against the architecture diagram in your head.

### Steps

1. Find how many nodes there are, and which one is the control plane.
2. List the Pods that make up the control plane.
3. Find one DaemonSet that runs on every node, and explain why.
4. Find the IP address of the API server endpoint.
5. Find which node is currently running the `metrics-server` Pod (if installed).

### Solution

```bash
# 1
kubectl get nodes
# Look at the ROLES column: one row will say `control-plane`.

# 2
kubectl get pods -n kube-system \
  --field-selector spec.nodeName=$(kubectl get nodes -l node-role.kubernetes.io/control-plane= -o name | cut -d/ -f2) \
  -o name | grep -E 'etcd|apiserver|controller-manager|scheduler'

# 3
kubectl get ds -n kube-system
# kube-proxy and the CNI (kindnet here) are DaemonSets so every node has them —
# kube-proxy implements Service routing, the CNI provides the Pod network.

# 4
kubectl cluster-info | grep -i 'control plane'

# 5
kubectl get pod -n kube-system -l k8s-app=metrics-server -o wide
```

### Verification

After running step 2 you should see exactly four Pod names (one of each:
`etcd-…`, `kube-apiserver-…`, `kube-controller-manager-…`, `kube-scheduler-…`).
For step 3, `kubectl get ds -n kube-system` should show `DESIRED == CURRENT == 3`
for `kube-proxy` and `kindnet` (matching the node count).
