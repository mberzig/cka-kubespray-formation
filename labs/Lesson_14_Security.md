# Lab 14 — RBAC: onboard a dev (User) and a CI robot (ServiceAccount)

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Every step is a `kubectl` command run
> with the cluster's `admin.conf`: **the RBAC mechanics are identical on any cluster**
> (also validated on `kind`). Substitute your own node/server names
> (`kubectl get nodes`, `kubectl config view`).

## Objectives

By the end of this lab you will be able, **as an administrator**, to:

1. Create a **human user** (`jane`) via a cluster-signed certificate, and build her a
   **kubeconfig**.
2. Grant her **read-only** access to the Pods of a namespace, and **prove** she can
   do nothing more.
3. Create a **ServiceAccount** (`ci-bot`) for a pipeline and grant it management of
   **Deployments**.
4. Reuse a **built-in ClusterRole** (`view`) instead of writing a Role.
5. **Audit** rights (`kubectl auth can-i --as / --list`).
6. **Diagnose and fix** a `Forbidden` error.

**Estimated time: 35–45 min.**

---

## Prerequisites

```bash
# You are admin on the cluster
kubectl auth can-i '*' '*'            # → yes

# Working namespace
kubectl create namespace dev
```

> 💡 Keep two terminals: one "admin" (your default kubeconfig) and one you will use
> to act **as** `jane` (Part 2). Otherwise, `--as` is enough to test everything
> without switching kubeconfig.

---

## Part 1 — The starting point: no identity has any rights

```bash
# The 'default' ServiceAccount of dev can do almost nothing
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:default
```

✅ **Expected:** `no`

> 🔑 Lesson #1: by default, **nothing is allowed**. You grant explicitly.

---

## Part 2 — Onboard a human: the user `jane`

### 2.1 — Generate a key + a certificate request (CSR)

```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane/O=dev-team"
```

- `CN=jane` → the **user name** seen by Kubernetes.
- `O=dev-team` → jane's **group**.

### 2.2 — Have the cluster sign the CSR

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(base64 -w0 < jane.csr)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 604800          # 7 days
  usages:
  - client auth
EOF

kubectl get csr jane
```

✅ **Expected:** the CSR appears with `CONDITION = Pending`.

> ℹ️ On macOS, `base64 -w0` does not exist → use `base64 | tr -d '\n'`.

### 2.3 — Approve and retrieve the signed certificate

```bash
kubectl certificate approve jane
kubectl get csr jane                       # CONDITION → Approved,Issued

kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
```

✅ **Verification:**

```bash
openssl x509 -in jane.crt -noout -subject
# subject=O=dev-team, CN=jane   (the exact order/format depends on the OpenSSL
#                                version; what matters: CN=jane and O=dev-team)
```

### 2.4 — Build jane's kubeconfig

```bash
# Grab the apiserver address and cluster CA from YOUR kubeconfig
CLUSTER=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')
SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
kubectl config view --minify --raw \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > ca.crt

# Write a kubeconfig dedicated to jane
kubectl config set-cluster "$CLUSTER" --server="$SERVER" \
  --certificate-authority=ca.crt --embed-certs=true \
  --kubeconfig=jane.kubeconfig
kubectl config set-credentials jane \
  --client-certificate=jane.crt --client-key=jane.key --embed-certs=true \
  --kubeconfig=jane.kubeconfig
kubectl config set-context jane --cluster="$CLUSTER" --user=jane --namespace=dev \
  --kubeconfig=jane.kubeconfig
kubectl config use-context jane --kubeconfig=jane.kubeconfig
```

✅ **Test — jane is authenticated but has NO rights:**

```bash
kubectl --kubeconfig=jane.kubeconfig get pods -n dev
```

**Expected:**
```text
Error from server (Forbidden): pods is forbidden: User "jane" cannot list
resource "pods" in API group "" in the namespace "dev"
```

> 🔑 Lesson #2: the certificate proves **who** she is (authentication), not what she
> is **allowed** to do (authorization). RBAC is missing.

---

## Part 3 — Grant `jane` read-only access to Pods

```bash
# 2. The Role (what is allowed)
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev

# 3. The RoleBinding (who is granted it)
kubectl create rolebinding jane-pod-reader \
  --role=pod-reader --user=jane -n dev
```

✅ **Verification (audit + real test):**

```bash
# Audit from the admin side
kubectl auth can-i list pods   --as=jane -n dev     # → yes
kubectl auth can-i delete pods --as=jane -n dev     # → no
kubectl auth can-i list pods   --as=jane -n default # → no  (the binding is in 'dev')

# Real test with jane's kubeconfig
kubectl --kubeconfig=jane.kubeconfig run web --image=nginx -n dev   # → Forbidden (create refused)
kubectl --kubeconfig=jane.kubeconfig get pods -n dev                # → OK (list, empty for now)
```

> 🔑 Lesson #3: the right is **bounded to the namespace** (`dev`) and **to the verb**
> (read, not create). That is least privilege in action.

---

## Part 4 — Grant an app access: the ServiceAccount `ci-bot`

A CI pipeline must **deploy** into `dev`. Same recipe, the identity changes.

```bash
# 1. Identity (an object, this time)
kubectl create serviceaccount ci-bot -n dev

# 2. Role: manage Deployments (apiGroup "apps")
kubectl create role deploy-manager \
  --verb=get,list,create,update,delete --resource=deployments -n dev

# 3. Binding to the ServiceAccount
kubectl create rolebinding cibot-deploy \
  --role=deploy-manager --serviceaccount=dev:ci-bot -n dev
```

✅ **Verification:**

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:dev:ci-bot -n dev        # → yes
kubectl auth can-i create secrets \
  --as=system:serviceaccount:dev:ci-bot -n dev        # → no
```

**Test "as the pipeline would", with the SA token.** Build a **dedicated** kubeconfig
that contains **only** the token:

```bash
TOKEN=$(kubectl create token ci-bot -n dev)
SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

kubectl config set-cluster lab --server="$SERVER" \
  --certificate-authority=ca.crt --embed-certs=true --kubeconfig=cibot.kubeconfig
kubectl config set-credentials ci-bot --token="$TOKEN" --kubeconfig=cibot.kubeconfig
kubectl config set-context ci-bot --cluster=lab --user=ci-bot --namespace=dev \
  --kubeconfig=cibot.kubeconfig
kubectl config use-context ci-bot --kubeconfig=cibot.kubeconfig

# Who am I with this token?
kubectl --kubeconfig=cibot.kubeconfig auth whoami
```

**Expected:**
```text
ATTRIBUTE   VALUE
Username    system:serviceaccount:dev:ci-bot
Groups      [system:serviceaccounts system:serviceaccounts:dev system:authenticated]
```

```bash
# Deploy as ci-bot (allowed) …
kubectl --kubeconfig=cibot.kubeconfig create deployment demo --image=nginx -n dev
#   → deployment.apps/demo created

# … but a Secret is refused (the SA boundary is proven)
kubectl --kubeconfig=cibot.kubeconfig create secret generic x --from-literal=a=b -n dev
#   → Error ... "system:serviceaccount:dev:ci-bot" cannot create resource "secrets" ...
```

> ⚠️ **Admin pitfall (important).** Do **not** use `kubectl --token=… --server=…` on
> top of your admin kubeconfig: on a cluster where the admin authenticates with a
> **client certificate** (kind, kubeadm, **Kubespray**), kubectl sends the certificate
> **in addition to** the token and the **certificate wins** — you stay admin without
> noticing (`auth whoami` would then show the admin's groups). The **dedicated**
> kubeconfig above avoids this trap.

> 🔑 Lesson #4: a **human** presents a certificate, an **app** presents a token — but
> the Role and the RoleBinding are written **exactly the same**.

---

## Part 5 — Reuse a built-in ClusterRole (don't reinvent)

Goal: jane should be able to **read everything** in `dev` (not just Pods). No need to
write a Role — the `view` ClusterRole already exists.

```bash
kubectl create rolebinding jane-view \
  --clusterrole=view --user=jane -n dev
```

✅ **Verification:**

```bash
kubectl auth can-i list services    --as=jane -n dev   # → yes (new)
kubectl auth can-i list deployments --as=jane -n dev   # → yes (new)
kubectl auth can-i get  secrets     --as=jane -n dev   # → no  ('view' excludes Secrets)
```

> 🔑 Lesson #5: a **RoleBinding** can point at a **ClusterRole** → you reuse a
> ready-made permission, but **scoped** to the single namespace `dev`.

---

## Part 6 — Diagnose and fix a "Forbidden"

A developer complains: *"my Pod can't list Pods from the API."* Let's reproduce and
fix it.

```bash
# The Pod runs with dev's 'default' SA → let's reproduce its right
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:default
```

**Expected:** `no` → **that's the gap**. Read the error as a checklist:

| The message would say… | You check |
|------------------------|-----------|
| `User "system:serviceaccount:dev:default"` | wrong identity → the Pod should have **its own SA** |
| `cannot **list** ... "pods"` | verb/resource missing from a Role |
| `namespace "dev"` | is the binding present in **this** namespace? |

**Clean fix** (dedicated SA + minimal right, rather than widening `default`):

```bash
kubectl create serviceaccount pod-lister -n dev
kubectl create role pod-list --verb=get,list --resource=pods -n dev
kubectl create rolebinding pod-lister-bind \
  --role=pod-list --serviceaccount=dev:pod-lister -n dev
```

✅ **Verification:**

```bash
kubectl auth can-i list pods -n dev \
  --as=system:serviceaccount:dev:pod-lister          # → yes
```

Then rewire the Pod onto this SA (`spec.serviceAccountName: pod-lister`).

---

## 🏆 Challenge (CKA exam style) — try it without looking at the solution

> Create a ServiceAccount **`monitor`** in namespace `monitoring` that can
> **read (get/list/watch)** **pods**, **services** and **nodes** across **ALL**
> namespaces. Prove it with `kubectl auth can-i`.

Hints: `nodes` is a **cluster-scoped** resource → you need a **ClusterRole** + a
**ClusterRoleBinding** (not a Role).

<details>
<summary>Solution</summary>

```bash
kubectl create namespace monitoring
kubectl create serviceaccount monitor -n monitoring

kubectl create clusterrole monitor-ro \
  --verb=get,list,watch --resource=pods,services,nodes

kubectl create clusterrolebinding monitor-ro-bind \
  --clusterrole=monitor-ro \
  --serviceaccount=monitoring:monitor

# Checks
kubectl auth can-i list nodes \
  --as=system:serviceaccount:monitoring:monitor                 # → yes (cluster-scoped)
kubectl auth can-i list pods -n kube-system \
  --as=system:serviceaccount:monitoring:monitor                 # → yes (all namespaces)
kubectl auth can-i delete pods \
  --as=system:serviceaccount:monitoring:monitor                 # → no
```

Why a ClusterRoleBinding and not a RoleBinding? A RoleBinding, even to a ClusterRole,
grants access only **within a single namespace** and **never** to cluster-scoped
resources like `nodes`.
</details>

---

## Verification summary

| # | Command | Expected |
|---|---------|----------|
| 1 | `auth can-i list pods --as=…:dev:default` | `no` |
| 2 | `get csr jane` after approve | `Approved,Issued` |
| 3 | `get pods` (jane kubeconfig, before RBAC) | `Forbidden` |
| 4 | `auth can-i list pods --as=jane -n dev` (after binding) | `yes` |
| 5 | `auth can-i delete pods --as=jane -n dev` | `no` |
| 6 | `auth can-i create deployments --as=…:dev:ci-bot -n dev` | `yes` |
| 7 | `auth whoami` with ci-bot's token | `system:serviceaccount:dev:ci-bot` |
| 8 | `auth can-i list services --as=jane -n dev` (after `view`) | `yes` |
| 9 | Challenge: `auth can-i list nodes --as=…:monitoring:monitor` | `yes` |

---

## Cleanup

```bash
kubectl delete namespace dev monitoring
kubectl delete csr jane
kubectl delete clusterrole monitor-ro
kubectl delete clusterrolebinding monitor-ro-bind
rm -f jane.key jane.csr jane.crt ca.crt jane.kubeconfig cibot.kubeconfig
```

> ⚠️ A client certificate **cannot be revoked** on the Kubernetes side: limit the
> risk with a **short duration** (`expirationSeconds`) and, in real production,
> prefer **OIDC** (centralized revocation + group management).

---

## What you take away

- **Identity → Role → RoleBinding**, and nothing works without the binding.
- **User = certificate** (human) · **ServiceAccount = object + token** (app); the
  RBAC recipe is **the same** afterwards.
- `Role`/`RoleBinding` = one namespace; `ClusterRole`/`ClusterRoleBinding` = the whole
  cluster (required for `nodes`, PV…).
- Reuse **`view`/`edit`/`admin`**; keep `cluster-admin` for emergencies.
- **`kubectl auth can-i [--as] [--list]`** is your stethoscope; the `Forbidden`
  message **is** the troubleshooting checklist.
