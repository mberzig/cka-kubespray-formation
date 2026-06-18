# Lesson 14 — Security Settings — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts

| Object               | Scope    | Use                                                      |
|----------------------|----------|----------------------------------------------------------|
| `ServiceAccount`     | namespace| Identity for Pods to authenticate to the API server.     |
| `Role`               | namespace| Set of `rules` (`apiGroups/resources/verbs`).            |
| `ClusterRole`        | cluster  | Same shape, applies cluster-wide or to cluster-scoped resources. |
| `RoleBinding`        | namespace| Binds subjects (User/Group/SA) to a Role.                |
| `ClusterRoleBinding` | cluster  | Binds subjects to a ClusterRole cluster-wide.            |
| `securityContext`    | pod/container | runAsUser, runAsNonRoot, capabilities, RO FS.       |

Verbs you'll commonly see: `get`, `list`, `watch`, `create`, `update`,
`patch`, `delete`, `deletecollection`.

---

## Examples

### 1. ServiceAccount + namespace-scoped Role + RoleBinding

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: {name: pod-reader, namespace: l10}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: pod-reader, namespace: l10}
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: pod-reader, namespace: l10}
subjects:
- {kind: ServiceAccount, name: pod-reader, namespace: l10}
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Test the permissions without rolling a Pod:

```bash
SA=system:serviceaccount:l10:pod-reader

kubectl -n l10 auth can-i list pods     --as=$SA      # yes
kubectl -n l10 auth can-i create pods   --as=$SA      # no
kubectl -n l10 auth can-i list secrets  --as=$SA      # no
kubectl -n other auth can-i list pods   --as=$SA      # no  (other ns not bound)
```

### 2. ClusterRole + ClusterRoleBinding (cluster-scoped resource)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: {name: node-viewer, namespace: l10}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: {name: node-viewer}
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: {name: node-viewer}
subjects:
- {kind: ServiceAccount, name: node-viewer, namespace: l10}
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
```

```bash
SA=system:serviceaccount:l10:node-viewer
kubectl auth can-i list nodes --as=$SA     # yes
kubectl auth can-i list pods  --as=$SA     # no
```

### 3. Pod uses the ServiceAccount + calls the API

```yaml
apiVersion: v1
kind: Pod
metadata: {name: kubectl-tool, namespace: l10}
spec:
  serviceAccountName: pod-reader
  containers:
  - name: kc
    image: bitnami/kubectl:latest
    command: ["sh","-c","sleep infinity"]
```

```bash
kubectl -n l10 exec kubectl-tool -- kubectl get pods -n l10     # OK
kubectl -n l10 exec kubectl-tool -- kubectl get pods -n default # forbidden
```

### 4. securityContext: non-root + drop capabilities + read-only FS

```yaml
apiVersion: v1
kind: Pod
metadata: {name: secure, namespace: l10}
spec:
  securityContext:                   # Pod-level
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","id; sleep infinity"]
    securityContext:                 # Container-level overrides
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
```

Observed behaviour:

```bash
kubectl -n l10 logs secure
# uid=10001 gid=0(root) groups=0(root),10001     ← not root user

kubectl -n l10 exec secure -- touch /etc/test
# touch: /etc/test: Read-only file system  ← FS write blocked
```

For applications that need write paths (web servers writing logs/PIDs), mount
`emptyDir` volumes at those paths or use an image purpose-built for non-root
(e.g. `nginxinc/nginx-unprivileged`).

### 5. Useful `auth` shortcuts

```bash
# What can the current user do?
kubectl auth can-i --list -n l10

# Impersonate
kubectl get pods --as=jane --as-group=devs -n l10
```

---

## Lab

**Goal**: build a `read-only-operator` ServiceAccount that can read every kind
of resource in namespace `team-a`, plus list nodes cluster-wide — and nothing
else. Run a Pod under that identity and prove the limits.

### Steps

1. Create namespace `team-a`.
2. Create ServiceAccount `read-only-operator` in `team-a`.
3. Give it the built-in ClusterRole **`view`** scoped to `team-a` (via a
   RoleBinding).
4. Add a ClusterRole `nodes-viewer` with `nodes: get/list/watch` and bind it
   cluster-wide to the SA.
5. Verify with `auth can-i`:
   - list pods in `team-a` → yes
   - create deployments in `team-a` → no
   - list nodes cluster-wide → yes
   - list pods in `default` → no
6. Launch a `bitnami/kubectl` Pod under the SA and confirm `kubectl get pods`
   works in `team-a` but fails in `default`.

### Solution

```bash
# 1
kubectl create ns team-a

# 2
kubectl -n team-a create serviceaccount read-only-operator

# 3
kubectl -n team-a create rolebinding view-in-team-a \
  --clusterrole=view \
  --serviceaccount=team-a:read-only-operator

# 4
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: {name: nodes-viewer}
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get","list","watch"]
EOF
kubectl create clusterrolebinding nodes-viewer-binding \
  --clusterrole=nodes-viewer \
  --serviceaccount=team-a:read-only-operator

# 5
SA=system:serviceaccount:team-a:read-only-operator
kubectl -n team-a   auth can-i list pods         --as=$SA      # yes
kubectl -n team-a   auth can-i create deployments --as=$SA     # no
kubectl             auth can-i list nodes         --as=$SA     # yes
kubectl -n default  auth can-i list pods         --as=$SA      # no

# 6  (kubectl run has no --serviceaccount flag — use a manifest)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: kctl, namespace: team-a}
spec:
  serviceAccountName: read-only-operator
  containers:
  - name: kc
    image: bitnami/kubectl:latest
    command: ["sh","-c","sleep infinity"]
EOF
kubectl -n team-a wait --for=condition=Ready pod/kctl --timeout=60s

kubectl -n team-a exec kctl -- kubectl get pods -n team-a       # OK (lists kctl itself)
kubectl -n team-a exec kctl -- kubectl get pods -n default      # Forbidden
kubectl -n team-a exec kctl -- kubectl get nodes                 # OK
```

### Verification

```
# Expected "auth can-i" outputs:
yes
no
yes
no
```

```
# Step 6 exec output (last command):
NAME                    STATUS   ROLES           AGE   VERSION
cka-lab-control-plane   Ready    control-plane   ...   v1.31.2
cka-lab-worker          Ready    <none>          ...   v1.31.2
cka-lab-worker2         Ready    <none>          ...   v1.31.2

# Forbidden ones look like:
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:team-a:read-only-operator"
cannot list resource "pods" in API group "" in the namespace "default"
```

### Cleanup

```bash
kubectl delete ns team-a
kubectl delete clusterrolebinding nodes-viewer-binding
kubectl delete clusterrole nodes-viewer
```
