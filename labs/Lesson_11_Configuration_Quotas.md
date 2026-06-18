# Lesson 11 — Configuration & Quotas — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts

| Object         | Purpose                                                      |
|----------------|--------------------------------------------------------------|
| **Namespace**  | Logical scope for resources, RBAC, quotas.                   |
| **ConfigMap**  | Non-sensitive key/value config injected as env or files.     |
| **Secret**     | Same shape as ConfigMap but base64'd, stored separately, can be encrypted at rest. |
| **ResourceQuota** | Namespace-wide caps (CPU/mem/objects).                   |
| **LimitRange** | Per-Container defaults & min/max in a namespace.             |

---

## Examples

### 1. ConfigMap from literals; consumed as environment

```bash
kubectl create ns l6
kubectl -n l6 create configmap app-cfg \
  --from-literal=ENV=prod --from-literal=LOG_LEVEL=info

kubectl -n l6 get cm app-cfg -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata: {name: cfg-consumer, namespace: l6}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","echo ENV=$ENV; echo LOG_LEVEL=$LOG_LEVEL; sleep infinity"]
    envFrom:
    - configMapRef: {name: app-cfg}
```

```bash
kubectl apply -f cfg-pod.yaml
kubectl -n l6 logs cfg-consumer    # ENV=prod, LOG_LEVEL=info
```

### 2. Secret consumed as a mounted file

```bash
kubectl -n l6 create secret generic db-cred \
  --from-literal=user=admin --from-literal=pwd=s3cret
```

```yaml
apiVersion: v1
kind: Pod
metadata: {name: secret-consumer, namespace: l6}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","cat /etc/secret/user; echo; cat /etc/secret/pwd; sleep infinity"]
    volumeMounts:
    - {name: sec, mountPath: /etc/secret, readOnly: true}
  volumes:
  - name: sec
    secret: {secretName: db-cred}
```

Each key in the Secret becomes a separate file: `/etc/secret/user`,
`/etc/secret/pwd`. Reading them prints `admin` and `s3cret`.

> The Secret store is base64 (not encrypted) by default. Production clusters
> enable encryption-at-rest in the apiserver `--encryption-provider-config`.

### 3. ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: {name: dev-quota, namespace: l6}
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    pods: "5"
```

Inspect usage live:

```bash
kubectl -n l6 describe resourcequota dev-quota
```

Trying to create a 6th Pod fails:

```
Error from server (Forbidden): pods "busy-5" is forbidden: exceeded quota: dev-quota,
requested: pods=1, used: pods=5, limited: pods=5
```

### 4. LimitRange — auto-inject default requests/limits

```yaml
apiVersion: v1
kind: LimitRange
metadata: {name: dev-default, namespace: l6}
spec:
  limits:
  - type: Container
    default:        {cpu: "200m", memory: "256Mi"}
    defaultRequest: {cpu: "100m", memory: "128Mi"}
```

A Pod created **without** `resources` inherits these:

```bash
kubectl -n l6 run noreq --image=nginx:1.27 --restart=Never
kubectl -n l6 get pod noreq -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}
```

Without LimitRange, the Pod would have **no** requests/limits — its scheduling
& OOM behaviour become unpredictable, and a ResourceQuota with `requests.cpu`
would reject it.

### 5. Switch the active namespace (avoid typing `-n ...`)

```bash
kubectl config set-context --current --namespace=l6
kubectl get pods             # implicit -n l6
kubectl config set-context --current --namespace=default
```

---

## Lab

**Goal**: deliver a multi-tenant namespace `dev` with a hard cap of 4 Pods,
2 CPU & 2 Gi requests, sensible per-Container defaults, plus a Pod that wires
config and a credential from a ConfigMap and Secret.

### Steps

1. Create namespace `dev`.
2. Create a LimitRange so any new Container gets defaultRequest `100m`/`128Mi`
   and default `500m`/`512Mi`.
3. Create a ResourceQuota: `pods=4`, `requests.cpu=2`, `requests.memory=2Gi`,
   `limits.cpu=4`, `limits.memory=4Gi`.
4. Create ConfigMap `app-config` with `LOG_LEVEL=debug` and a key
   `app.properties` containing `feature.x=true\nfeature.y=false`.
5. Create Secret `app-secret` with `api_key=ABCD-1234`.
6. Create Deployment `webapp` (1 replica, `nginx:1.27`) that:
   - Sets env `LOG_LEVEL` from the CM.
   - Mounts `app-config` key `app.properties` as `/etc/app/app.properties`.
   - Mounts the Secret at `/etc/app/secret/`.

### Solution

```bash
# 1
kubectl create ns dev

# 2
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata: {name: defaults, namespace: dev}
spec:
  limits:
  - type: Container
    default:        {cpu: "500m", memory: "512Mi"}
    defaultRequest: {cpu: "100m", memory: "128Mi"}
EOF

# 3
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata: {name: cap, namespace: dev}
spec:
  hard:
    pods: "4"
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
EOF

# 4
kubectl -n dev create configmap app-config \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=app.properties=$'feature.x=true\nfeature.y=false'

# 5
kubectl -n dev create secret generic app-secret --from-literal=api_key=ABCD-1234

# 6
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: webapp, namespace: dev}
spec:
  replicas: 1
  selector: {matchLabels: {app: webapp}}
  template:
    metadata: {labels: {app: webapp}}
    spec:
      containers:
      - name: web
        image: nginx:1.27
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef: {name: app-config, key: LOG_LEVEL}
        volumeMounts:
        - {name: cfg, mountPath: /etc/app/app.properties, subPath: app.properties}
        - {name: sec, mountPath: /etc/app/secret, readOnly: true}
      volumes:
      - name: cfg
        configMap:
          name: app-config
          items: [{key: app.properties, path: app.properties}]
      - name: sec
        secret: {secretName: app-secret}
EOF
```

### Verification

```bash
kubectl -n dev describe resourcequota cap
kubectl -n dev get limitrange defaults -o yaml
kubectl -n dev get cm app-config -o yaml
kubectl -n dev get secret app-secret -o yaml

POD=$(kubectl -n dev get pod -l app=webapp -o name | head -1)
kubectl -n dev exec $POD -- env | grep LOG_LEVEL
kubectl -n dev exec $POD -- cat /etc/app/app.properties
kubectl -n dev exec $POD -- cat /etc/app/secret/api_key
```

Expected:

- `LOG_LEVEL=debug`
- `app.properties` content: `feature.x=true\nfeature.y=false`
- `api_key`: `ABCD-1234`
- Container resources set automatically: requests `100m/128Mi`, limits
  `500m/512Mi`.

### Cleanup

```bash
kubectl delete ns dev
```
