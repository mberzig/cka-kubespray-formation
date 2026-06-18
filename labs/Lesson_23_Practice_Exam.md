# Lesson 23 — Practice Exam — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Practice Exam #1

### Task 1.1 — Build & expose

In namespace `pe1-app`:

1. Create a Deployment `web` with image `nginx:1.27`, **3 replicas**,
   label `tier=frontend`.
2. Expose it as a ClusterIP Service `web` on port 80.
3. Create a NetworkPolicy that **only** allows ingress to `web` from Pods
   labelled `role=tester` in the same namespace.

### Task 1.2 — Storage

In namespace `pe1-storage`:

1. Create a PVC `data` of 200 Mi using the default StorageClass.
2. Run a Pod `writer` (image `busybox:1.37`) mounting the PVC at `/data` that
   writes `done` into `/data/marker` and sleeps.
3. After writer is `Ready`, delete it, create a Pod `reader` reading
   `/data/marker` and printing it to stdout. Verify the file value persists.

### Task 1.3 — Scheduling

1. Label `cka-lab-worker` with `tier=gold`.
2. Taint `cka-lab-worker2` with `dedicated=batch:NoSchedule`.
3. Create a Pod `payments` (`nginx:1.27`) in namespace `pe1-sched` that:
   - must run on a `tier=gold` node;
   - tolerates the `dedicated=batch:NoSchedule` taint;
   - has resource requests `cpu=100m, memory=128Mi` and limits
     `cpu=500m, memory=512Mi`.

### Task 1.4 — Troubleshooting

A new namespace `pe1-broken` has been created with one Deployment `broken`
that won't roll out. Diagnose, fix it, and make it `1/1 Available` with
**minimum** YAML changes.

Setup (run first):

```bash
kubectl create ns pe1-broken
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: broken, namespace: pe1-broken}
spec:
  replicas: 1
  selector: {matchLabels: {app: broken}}
  template:
    metadata: {labels: {app: broken}}
    spec:
      nodeSelector: {disktype: nvme}     # bug: no node labelled nvme
      containers:
      - name: app
        image: nginx:1.27
        resources:
          requests: {cpu: "8",    memory: "16Gi"}    # bug: too big
          limits:   {cpu: "8",    memory: "16Gi"}
EOF
```

### Solution #1

```bash
# 1.1
kubectl create ns pe1-app
kubectl -n pe1-app create deployment web --image=nginx:1.27 --replicas=3
kubectl -n pe1-app label deployment web tier=frontend
kubectl -n pe1-app patch deployment web --type=merge -p \
  '{"spec":{"template":{"metadata":{"labels":{"tier":"frontend"}}}}}'
kubectl -n pe1-app expose deployment web --name=web --port=80
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: only-testers, namespace: pe1-app}
spec:
  podSelector: {matchLabels: {tier: frontend}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {matchLabels: {role: tester}}
    ports: [{port: 80, protocol: TCP}]
EOF

# 1.2
kubectl create ns pe1-storage
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data, namespace: pe1-storage}
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 200Mi}}
---
apiVersion: v1
kind: Pod
metadata: {name: writer, namespace: pe1-storage}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","echo done > /data/marker && tail -f /dev/null"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: data}}]
EOF
kubectl -n pe1-storage wait --for=condition=Ready pod/writer --timeout=60s
kubectl -n pe1-storage delete pod writer
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: reader, namespace: pe1-storage}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","cat /data/marker; tail -f /dev/null"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: data}}]
EOF
kubectl -n pe1-storage wait --for=condition=Ready pod/reader --timeout=60s
kubectl -n pe1-storage logs reader      # must print "done"

# 1.3
kubectl label node cka-lab-worker  tier=gold --overwrite
kubectl taint nodes cka-lab-worker2 dedicated=batch:NoSchedule --overwrite
kubectl create ns pe1-sched
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: payments, namespace: pe1-sched}
spec:
  nodeSelector: {tier: gold}
  tolerations:
  - {key: dedicated, operator: Equal, value: batch, effect: NoSchedule}
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests: {cpu: "100m", memory: "128Mi"}
      limits:   {cpu: "500m", memory: "512Mi"}
EOF

# 1.4 — diagnose then fix
kubectl -n pe1-broken describe pod -l app=broken | grep -A2 'Events:'
# Fix 1: remove unsatisfiable nodeSelector
kubectl -n pe1-broken patch deployment broken --type=json -p='[
  {"op":"remove","path":"/spec/template/spec/nodeSelector"}
]'
# Fix 2: shrink the impossibly-large resources
kubectl -n pe1-broken patch deployment broken --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/resources","value":{"requests":{"cpu":"100m","memory":"128Mi"},"limits":{"cpu":"500m","memory":"512Mi"}}}
]'
kubectl -n pe1-broken rollout status deployment/broken
```

### Verify #1

```bash
kubectl -n pe1-app get deploy,svc,netpol
kubectl -n pe1-storage logs reader               # "done"
kubectl -n pe1-sched get pod payments -o wide    # on cka-lab-worker (or wherever tier=gold)
kubectl -n pe1-broken get pod                    # broken-* Running
```

### Cleanup #1

```bash
kubectl delete ns pe1-app pe1-storage pe1-sched pe1-broken
kubectl taint nodes cka-lab-worker2 dedicated=batch:NoSchedule-
kubectl label node cka-lab-worker tier-
```

---

## Practice Exam #2

### Task 2.1 — RBAC & ServiceAccount

In namespace `pe2-rbac`:

1. Create a ServiceAccount `auditor`.
2. Give it permission to **list & watch** every resource in `pe2-rbac` and
   in `pe2-app` (but no other namespace, and **no write** verbs).
3. Run a Pod `auditor-pod` as that SA (image `bitnami/kubectl:latest`).
   Verify it can list pods in both namespaces but not in `default`.

### Task 2.2 — etcd snapshot

Take a snapshot of etcd named `/tmp/exam-snap.db` inside the etcd Pod and
verify it with `snapshot status`.

### Task 2.3 — Node drain

`cka-lab-worker2` needs maintenance. Cordon and drain it (ignore DaemonSets,
delete emptyDir data). Confirm all user Pods have moved off it. Uncordon
afterwards.

### Task 2.4 — Ingress

In namespace `pe2-ing`:

1. Deploy `echo` (`hashicorp/http-echo:1.0`, 2 replicas, args `-text=v2`,
   listen `:5678`).
2. Expose it as Service `echo` on port 80 → 5678.
3. Create an Ingress that routes `Host: echo.local` to that Service via the
   `nginx` ingress class.
4. Verify by `curl -H 'Host: echo.local'` to the controller's Pod IP.

### Solution #2

```bash
# 2.1
kubectl create ns pe2-rbac
kubectl create ns pe2-app

kubectl -n pe2-rbac create serviceaccount auditor
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: {name: read-only-auditor}
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get","list","watch"]
EOF

# Bind it in BOTH namespaces only (so default and others remain off-limits)
for ns in pe2-rbac pe2-app; do
  kubectl -n $ns create rolebinding auditor-in-$ns \
    --clusterrole=read-only-auditor \
    --serviceaccount=pe2-rbac:auditor
done

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: auditor-pod, namespace: pe2-rbac}
spec:
  serviceAccountName: auditor
  containers:
  - name: kc
    image: bitnami/kubectl:latest
    command: ["sh","-c","sleep infinity"]
EOF
kubectl -n pe2-rbac wait --for=condition=Ready pod/auditor-pod --timeout=120s
kubectl -n pe2-rbac exec auditor-pod -- kubectl get pod -n pe2-rbac
kubectl -n pe2-rbac exec auditor-pod -- kubectl get pod -n pe2-app
kubectl -n pe2-rbac exec auditor-pod -- kubectl get pod -n default   # Forbidden

# 2.2
ETCD_POD=$(kubectl -n kube-system get pod -l component=etcd -o name | head -1)
kubectl -n kube-system exec $ETCD_POD -- sh -c "
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/exam-snap.db && \
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /tmp/exam-snap.db"

# 2.3
kubectl cordon cka-lab-worker2
kubectl drain  cka-lab-worker2 --ignore-daemonsets --delete-emptydir-data --force
kubectl get pod -A -o wide | awk '$8=="cka-lab-worker2"'      # only kube-system DS pods
kubectl uncordon cka-lab-worker2

# 2.4
kubectl create ns pe2-ing
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: echo, namespace: pe2-ing}
spec:
  replicas: 2
  selector: {matchLabels: {app: echo}}
  template:
    metadata: {labels: {app: echo}}
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args: ["-listen=:5678", "-text=v2"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata: {name: echo, namespace: pe2-ing}
spec:
  selector: {app: echo}
  ports: [{port: 80, targetPort: 5678}]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: echo-ing, namespace: pe2-ing}
spec:
  ingressClassName: nginx
  rules:
  - host: echo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: echo
            port: {number: 80}
EOF
kubectl -n pe2-ing rollout status deployment/echo
INGRESS_IP=$(kubectl -n ingress-nginx get pod \
  -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].status.podIP}')
kubectl run check --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  curl -s -H 'Host: echo.local' http://$INGRESS_IP/
# Expected: v2
```

### Verify #2

```bash
kubectl -n pe2-rbac exec auditor-pod -- kubectl get pod -n pe2-app    # OK
kubectl -n pe2-rbac exec auditor-pod -- kubectl get pod -n default    # Forbidden
# etcd snapshot status printed HASH/REVISION/TOTAL KEYS/TOTAL SIZE row
# After uncordon, cka-lab-worker2 is Ready (no SchedulingDisabled)
kubectl -n pe2-ing get ingress    # HOSTS=echo.local, CLASS=nginx, ADDRESS populated
# curl through ingress returned "v2"
```

### Cleanup #2

```bash
kubectl delete ns pe2-rbac pe2-app pe2-ing
kubectl delete clusterrole read-only-auditor
```

---

## Time-saving habits

- Aliases — `alias k=kubectl`, `export do='--dry-run=client -o yaml'`.
- `kubectl explain pod.spec.containers.resources` — schema reference offline.
- Generate manifests fast: `kubectl create deploy nginx --image=nginx:1.27 $do > deploy.yaml`.
- Patch instead of edit when the change is small: `kubectl patch deploy ... --type=json -p='[...]'`.
- `kubectl get pod -o wide` shows IP/node; `kubectl get pod -o yaml | less` shows everything.
- `kubectl auth can-i --as=...` to verify RBAC without spinning a Pod.
