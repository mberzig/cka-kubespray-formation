# Lesson 8 — Deploying Applications & Autoscaling — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts

| Workload     | Use case                                                                        |
|--------------|---------------------------------------------------------------------------------|
| **Pod**      | Smallest deployable unit. Almost never created directly — wrap it.              |
| **Deployment** | Stateless apps, rolling updates, easy scale.                                  |
| **DaemonSet**  | One Pod **per node** (log collector, CNI, node-exporter).                     |
| **StatefulSet**| Stable identity & per-replica storage (databases, etcd-like apps).            |
| **Job/CronJob**| Run-to-completion / scheduled batch work.                                      |

A Pod can run multiple containers: an **init container** runs to completion
before the main app starts; **sidecar containers** run alongside the main app
(logging, proxies).

---

## Examples

### 1. Imperative Deployment + scale + image update

```bash
kubectl create namespace l3
kubectl -n l3 create deployment nginx --image=nginx:1.27 --replicas=3
kubectl -n l3 rollout status deployment/nginx
kubectl -n l3 scale deployment nginx --replicas=5
kubectl -n l3 set image deployment/nginx nginx=nginx:1.28
kubectl -n l3 rollout history deployment/nginx
kubectl -n l3 rollout undo deployment/nginx           # rollback to previous revision
```

### 2. DaemonSet — one Pod per node

```yaml
# ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: {name: node-tool, namespace: l3}
spec:
  selector: {matchLabels: {app: node-tool}}
  template:
    metadata: {labels: {app: node-tool}}
    spec:
      containers:
      - name: tool
        image: busybox:1.37
        command: ["sh","-c","while true; do echo $(hostname): $(date); sleep 30; done"]
```

```bash
kubectl apply -f ds.yaml
kubectl -n l3 get pod -l app=node-tool -o wide      # exactly N pods, one per worker node
```

### 3. StatefulSet with per-replica persistent storage

```yaml
# sts.yaml
apiVersion: v1
kind: Service
metadata: {name: web, namespace: l3}
spec:
  clusterIP: None                  # headless service — required for STS DNS
  selector: {app: web}
  ports: [{port: 80, name: web}]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: web, namespace: l3}
spec:
  serviceName: web
  replicas: 2
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports: [{containerPort: 80, name: web}]
        volumeMounts: [{name: data, mountPath: /usr/share/nginx/html}]
  volumeClaimTemplates:
  - metadata: {name: data}
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: {requests: {storage: 100Mi}}
```

```bash
kubectl apply -f sts.yaml
kubectl -n l3 get sts,pvc,pod -l app=web    # web-0, web-1 with stable names; PVCs named data-web-0/1
```

### 4. Pod with init container and sidecar

```yaml
# pod-init.yaml
apiVersion: v1
kind: Pod
metadata: {name: app-with-init, namespace: l3}
spec:
  initContainers:
  - name: setup
    image: busybox:1.37
    command: ["sh","-c","echo 'Hello from init at '$(date) > /shared/index.html"]
    volumeMounts: [{name: shared, mountPath: /shared}]
  containers:
  - name: web
    image: nginx:1.27
    volumeMounts: [{name: shared, mountPath: /usr/share/nginx/html}]
  - name: log-sidecar
    image: busybox:1.37
    command: ["sh","-c","tail -F /usr/share/nginx/html/index.html"]
    volumeMounts: [{name: shared, mountPath: /usr/share/nginx/html}]
  volumes:
  - name: shared
    emptyDir: {}
```

```bash
kubectl apply -f pod-init.yaml
kubectl -n l3 exec app-with-init -c web -- cat /usr/share/nginx/html/index.html
kubectl -n l3 logs app-with-init -c log-sidecar
```

Observed output:

```
Hello from init at Thu May 21 21:22:30 UTC 2026
```

The init container ran first, wrote the file, and exited. The `web` and
`log-sidecar` containers then started together and share the `emptyDir` volume.

---

## Lab

**Goal**: ship a small web app with controlled rollout, a DaemonSet, and a
StatefulSet — all in one namespace.

### Steps

1. Create namespace `l3-lab`.
2. Create a Deployment `web` with image `nginx:1.27`, 3 replicas, label
   `tier=frontend`.
3. Update the Deployment to `nginx:1.28` and verify the rollout completes.
4. Scale it up to 5 replicas, then back to 2.
5. Roll the Deployment back to revision 1.
6. Create a DaemonSet `agent` (busybox sleeping) — should run on every worker
   node (not control plane).
7. Create a StatefulSet `data` of 2 replicas with a 50 Mi per-replica PVC.
8. Verify: pods `data-0` and `data-1` exist and each has its own PVC.

### Solution

```bash
# 1
kubectl create ns l3-lab

# 2
kubectl -n l3-lab create deployment web --image=nginx:1.27 --replicas=3
kubectl -n l3-lab label deployment web tier=frontend
# Also propagate the label to the pod template:
kubectl -n l3-lab patch deployment web --type=merge \
  -p '{"spec":{"template":{"metadata":{"labels":{"tier":"frontend"}}}}}'

# 3
kubectl -n l3-lab set image deployment/web nginx=nginx:1.28
kubectl -n l3-lab rollout status deployment/web

# 4
kubectl -n l3-lab scale deployment/web --replicas=5
kubectl -n l3-lab scale deployment/web --replicas=2

# 5
kubectl -n l3-lab rollout undo deployment/web --to-revision=1

# 6
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata: {name: agent, namespace: l3-lab}
spec:
  selector: {matchLabels: {app: agent}}
  template:
    metadata: {labels: {app: agent}}
    spec:
      containers:
      - name: agent
        image: busybox:1.37
        command: ["sh","-c","sleep infinity"]
EOF

# 7
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata: {name: data, namespace: l3-lab}
spec:
  clusterIP: None
  selector: {app: data}
  ports: [{port: 80, name: data}]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: data, namespace: l3-lab}
spec:
  serviceName: data
  replicas: 2
  selector: {matchLabels: {app: data}}
  template:
    metadata: {labels: {app: data}}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        volumeMounts: [{name: data, mountPath: /usr/share/nginx/html}]
  volumeClaimTemplates:
  - metadata: {name: data}
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: {requests: {storage: 50Mi}}
EOF
```

### Verification

```bash
kubectl -n l3-lab get deploy,ds,sts
kubectl -n l3-lab get pvc
kubectl -n l3-lab rollout history deployment/web
kubectl -n l3-lab get pod -o wide      # DaemonSet on 2 workers, STS pods data-0 and data-1
```

Expected: deployment back to `nginx:1.27` (image, after `rollout undo`); 2
DaemonSet Pods (one per worker); 2 StatefulSet Pods + 2 PVCs named
`data-data-0` and `data-data-1`.

> ☁️ **On the OpenStack lab cluster** the PVCs bind via the default `cinder-csi`
> StorageClass (Cinder/Ceph). Cinder has a **1 GiB minimum volume size**, so the
> `50Mi`/`100Mi` requests above show up **Bound at `1Gi`** in `kubectl get pvc` —
> expected, not an error. The DaemonSet lands on the 2 workers only (the control
> planes carry the `node-role.kubernetes.io/control-plane:NoSchedule` taint).

### Cleanup

```bash
kubectl delete ns l3-lab
```

---

## Concepts

- The **HorizontalPodAutoscaler (HPA)** automatically adjusts the **replica
  count** of a Deployment/ReplicaSet/StatefulSet based on observed metrics
  (CPU, memory, or custom/external).
- HPA **requires the metrics-server** (or an adapter) and the target Pods must
  declare resource **`requests`** — `Utilization` is a percentage of the request.
- It does **not** apply to DaemonSets (their count tracks nodes).
- Replica math: `desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)`,
  bounded by `minReplicas`/`maxReplicas`.
- Siblings (separate add-ons): **VPA** tunes Pod requests/limits; **Cluster
  Autoscaler** adds/removes nodes. Don't pair HPA and VPA on the *same* metric.

## Setup (one-time)

> ℹ️ The combined formation provisions metrics-server through Kubespray in **L19**
> (`metrics_server_enabled: true`). If it isn't enabled yet, install it standalone:

```bash
# metrics-server — needs --kubelet-insecure-tls on BOTH kind AND the Kubespray/OpenStack
# lab cluster (kubelet serving certs are self-signed unless cert rotation is enabled).
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl -n kube-system patch deploy metrics-server --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl -n kube-system rollout status deploy/metrics-server
kubectl top nodes      # should return CPU/memory once ready
```

---

## Examples

### 1. Deploy a CPU-bound app with resource requests

```bash
kubectl create ns hpa-demo
kubectl -n hpa-demo create deployment php-apache --image=registry.k8s.io/hpa-example
kubectl -n hpa-demo set resources deployment php-apache --requests=cpu=200m --limits=cpu=500m
kubectl -n hpa-demo expose deployment php-apache --port=80
kubectl -n hpa-demo wait --for=condition=Available deployment/php-apache --timeout=300s
```

> Requests are mandatory for `Utilization`-based HPA — without `requests.cpu`,
> the HPA reports `TARGETS: <unknown>` and never scales.

### 2. Create the HorizontalPodAutoscaler

Declarative (`autoscaling/v2`):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: php-apache, namespace: hpa-demo}
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: php-apache}
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: {type: Utilization, averageUtilization: 50}
```

Or imperatively:

```bash
kubectl -n hpa-demo autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl -n hpa-demo get hpa php-apache
```

Sample output (idle):

```
NAME         REFERENCE               TARGETS              MINPODS   MAXPODS   REPLICAS
php-apache   Deployment/php-apache   cpu: <unknown>/50%   1         10        1
```

### 3. Generate load and watch it scale out

```bash
# in-cluster load generator
kubectl -n hpa-demo run load --image=busybox:1.37 --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache >/dev/null; done"

# watch the HPA react (Ctrl-C to stop)
kubectl -n hpa-demo get hpa php-apache -w
```

**Captured progression** (real run):

```
TARGETS              REPLICAS
cpu: <unknown>/50%   1
cpu: 250%/50%        1        # load hits; CPU far over target
cpu: 164%/50%        4        # scaled up
cpu: 84%/50%         7
cpu: 36%/50%         7        # stabilised under target at 7 replicas
```

The HPA scaled `php-apache` **1 → 4 → 7** as CPU spiked to 250%, then held at 7
once utilisation dropped back under 50%.

### 4. Remove the load → scale back in

```bash
kubectl -n hpa-demo delete pod load
# After the stabilisation window (~5 min by default), REPLICAS returns toward 1.
kubectl -n hpa-demo get hpa php-apache
```

> Scale-down is deliberately slow (`--horizontal-pod-autoscaler-downscale-stabilization`,
> default **5 min**) to avoid flapping.

---

## Lab

**Goal**: autoscale a Deployment on CPU and observe a real scale-out under load.

### Steps

1. In namespace `hpa-lab`, deploy `web` from `registry.k8s.io/hpa-example` with
   `requests.cpu=100m`, `limits.cpu=500m`; expose it on port 80.
2. Create an HPA: target **50%** CPU, **min 2**, **max 8** (use the
   `autoscaling/v2` manifest).
3. Generate load from an in-cluster Pod; watch `kubectl get hpa -w` until
   `REPLICAS` rises above 2.
4. Stop the load; confirm the HPA reports CPU back under target.

### Solution

```bash
kubectl create ns hpa-lab
kubectl -n hpa-lab create deployment web --image=registry.k8s.io/hpa-example
kubectl -n hpa-lab set resources deployment web --requests=cpu=100m --limits=cpu=500m
kubectl -n hpa-lab expose deployment web --port=80
kubectl -n hpa-lab wait --for=condition=Available deployment/web --timeout=300s

cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: web, namespace: hpa-lab}
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: web}
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource: {name: cpu, target: {type: Utilization, averageUtilization: 50}}
EOF

kubectl -n hpa-lab run load --image=busybox:1.37 --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://web >/dev/null; done"
kubectl -n hpa-lab get hpa web -w
```

### Verification

```bash
kubectl -n hpa-lab get hpa web      # TARGETS climbs past 50%, REPLICAS > 2 (up to 8)
kubectl -n hpa-lab get deploy web   # replica count matches the HPA's currentReplicas
```

Expected: under load `TARGETS` exceeds `50%` and `REPLICAS` scales out toward the
`maxReplicas` (8); after `kubectl -n hpa-lab delete pod load`, CPU falls back under
target and (after the stabilisation window) replicas trend back toward `minReplicas` (2).

### Cleanup

```bash
kubectl delete ns hpa-lab
```
