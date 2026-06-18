# Lesson 9 — Managing Storage — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts

| Object             | Role                                                                 |
|--------------------|----------------------------------------------------------------------|
| **Volume**         | A directory mounted into a Pod (`emptyDir`, `hostPath`, PVC…).       |
| **PersistentVolume (PV)** | Cluster-scoped storage resource (NFS share, EBS volume, hostPath…). |
| **PersistentVolumeClaim (PVC)** | Namespaced request for storage; matched against an available PV. |
| **StorageClass (SC)** | Template for **dynamic** provisioning (provisioner + parameters). |

A Pod references a PVC, the PVC binds to a PV, the PV references the underlying
storage. Dynamic provisioning creates the PV automatically from a StorageClass.

---

## Examples

### 1. Inspect the cluster's StorageClass

```bash
kubectl get sc
```

Expected (kind):

```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false
```

`(default)` means PVCs without an explicit `storageClassName` will use it.

### 2. Dynamic PVC + Pod that uses it

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data, namespace: l4}
spec:
  accessModes: ["ReadWriteOnce"]
  resources: {requests: {storage: 100Mi}}
  storageClassName: standard
---
apiVersion: v1
kind: Pod
metadata: {name: writer, namespace: l4}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","echo Persistent-$(date +%s) > /data/file && tail -f /dev/null"]
    volumeMounts:
    - {name: data, mountPath: /data}
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data
```

```bash
kubectl create ns l4
kubectl apply -f pvc.yaml
kubectl -n l4 get pv,pvc,pod
kubectl -n l4 exec writer -- cat /data/file       # prints Persistent-<timestamp>
```

Note: with `WaitForFirstConsumer`, the PV is only created once a Pod scheduled
on a node references the PVC.

### 3. Static PV pinned to a specific node

```yaml
# static.yaml
apiVersion: v1
kind: PersistentVolume
metadata: {name: static-pv}
spec:
  capacity: {storage: 50Mi}
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual           # any name not used by a SC
  hostPath: {path: /tmp/static-pv}
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values: [cka-lab-worker]
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: static-claim, namespace: l4}
spec:
  accessModes: ["ReadWriteOnce"]
  resources: {requests: {storage: 50Mi}}
  storageClassName: manual
```

`kubectl get pv` shows the PV bound to `l4/static-claim`. Without setting
`storageClassName: manual` on both sides, the PVC would try to be served by the
*default* SC and never match this hand-rolled PV.

### 4. emptyDir + ConfigMap volume

```yaml
apiVersion: v1
kind: Pod
metadata: {name: multi-vol}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","sleep infinity"]
    volumeMounts:
    - {name: scratch, mountPath: /scratch}
    - {name: config,  mountPath: /etc/config}
  volumes:
  - name: scratch
    emptyDir: {}                    # ephemeral, deleted with the Pod
  - name: config
    configMap:
      name: app-config              # must exist beforehand
```

### 5. Access modes recap

| Mode  | Meaning                                              |
|-------|------------------------------------------------------|
| RWO   | ReadWriteOnce — one **node** can mount RW.            |
| ROX   | ReadOnlyMany — many nodes mount RO.                   |
| RWX   | ReadWriteMany — many nodes mount RW (NFS, CephFS…).   |
| RWOP  | ReadWriteOncePod — only ONE pod can mount RW.         |

---

## Lab

**Goal**: Use both dynamic and static provisioning, observe binding, and prove
the data survives Pod restarts.

### Steps

1. Create namespace `l4-lab`.
2. Create a 200 Mi PVC `app-data` using the default StorageClass.
3. Create a Pod `writer` that writes `Hello-<random>` into `/data/marker` and
   sleeps.
4. Delete the Pod. Create a new Pod `reader` referencing the same PVC, and
   confirm `/data/marker` still contains the same value.
5. Create a static PV `archive-pv` of 100 Mi (storageClass `archive`, hostPath
   `/tmp/archive`, RWO, Retain). Bind a PVC `archive-claim` to it.

### Solution

```bash
# 1
kubectl create ns l4-lab

# 2
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: app-data, namespace: l4-lab}
spec:
  accessModes: ["ReadWriteOnce"]
  resources: {requests: {storage: 200Mi}}
EOF

# 3
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: writer, namespace: l4-lab}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","echo Hello-$RANDOM > /data/marker && tail -f /dev/null"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes:
  - {name: d, persistentVolumeClaim: {claimName: app-data}}
EOF
kubectl -n l4-lab wait --for=condition=Ready pod/writer --timeout=60s
MARKER=$(kubectl -n l4-lab exec writer -- cat /data/marker)
echo "Wrote: $MARKER"

# 4
kubectl -n l4-lab delete pod writer
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: reader, namespace: l4-lab}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","cat /data/marker; tail -f /dev/null"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes:
  - {name: d, persistentVolumeClaim: {claimName: app-data}}
EOF
kubectl -n l4-lab wait --for=condition=Ready pod/reader --timeout=60s
kubectl -n l4-lab exec reader -- cat /data/marker      # must equal $MARKER

# 5
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata: {name: archive-pv}
spec:
  capacity: {storage: 100Mi}
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: archive
  hostPath: {path: /tmp/archive}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: archive-claim, namespace: l4-lab}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: archive
  resources: {requests: {storage: 100Mi}}
EOF
```

### Verification

```bash
kubectl -n l4-lab get pv,pvc
# - app-data PVC: Bound to a dynamically-created PV from the DEFAULT SC.
# - archive-claim PVC: Bound to archive-pv.

# data persistence: step 4's `cat /data/marker` output must match the writer's output.
```

> ☁️ **On the OpenStack lab cluster** the default StorageClass is **`cinder-csi`**
> (`cinder.csi.openstack.org`, backed by Ceph) — not kind's `standard`/`local-path`.
> Cinder enforces a **1 GiB minimum**, so `app-data` (200Mi) binds at **`1Gi`**;
> the static `archive-pv` is a `hostPath` PV so it stays exactly `100Mi`.
> Data persistence still holds: the Cinder RWO volume detaches from `writer`'s node
> and re-attaches to `reader`'s node.

### Cleanup

```bash
kubectl delete ns l4-lab
kubectl delete pv archive-pv
```
