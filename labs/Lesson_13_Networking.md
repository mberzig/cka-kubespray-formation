# Lesson 13 — Networking & NetworkPolicy — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts

- **CNI plugin** — implements the Pod network. Examples: Calico, Cilium,
  Flannel, kindnet. Pods get a flat L3 address space.
- **kube-proxy** — programs the host's iptables/IPVS so Service ClusterIPs
  load-balance to backing Pods.
- **CoreDNS** — cluster DNS at `kube-dns.kube-system` (`10.96.0.10` here).
- **NetworkPolicy** — namespaced firewall rules that select Pods by label and
  declare allowed ingress/egress sources.

DNS pattern again:

```
<svc>.<ns>.svc.cluster.local
```

---

## Examples

### 1. Inspect the CNI

```bash
kubectl get pod -n kube-system -l app=kindnet \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

### 2. Resolve a Service via DNS

```bash
kubectl run dns --image=busybox:1.37 --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local
```

Returns `10.96.0.1` (the apiserver Service ClusterIP).

### 3. Deny-all NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: deny-all-ingress, namespace: l9}
spec:
  podSelector: {}                  # selects every Pod in the namespace
  policyTypes: [Ingress]
  # no `ingress:` block → no rule → all ingress denied
```

Without this policy, the `web` Pod is reachable. After applying, any cross-Pod
HTTP gets `download timed out`.

### 4. Allow only Pods carrying `role=trusted`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: allow-from-trusted, namespace: l9}
spec:
  podSelector: {matchLabels: {app: web}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {matchLabels: {role: trusted}}
    ports: [{port: 80, protocol: TCP}]
```

```bash
# untrusted — denied (timeout)
kubectl -n l9 run untrusted --image=busybox:1.37 --rm -it --restart=Never -- \
  wget -qO- -T 3 http://web/

# trusted — allowed
kubectl -n l9 run trusted --image=busybox:1.37 --rm -it --restart=Never \
  --labels=role=trusted -- wget -qO- -T 3 http://web/
```

### 5. Allow traffic from another namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: allow-monitoring, namespace: l9}
spec:
  podSelector: {matchLabels: {app: web}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels: {role: monitoring}
```

Requires the source namespace to have the matching label:

```bash
kubectl label namespace monitoring role=monitoring
```

### 6. Egress policy — restrict outbound

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: egress-to-dns-only, namespace: l9}
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: kube-system}}
      podSelector: {matchLabels: {k8s-app: kube-dns}}
    ports:
    - {protocol: UDP, port: 53}
    - {protocol: TCP, port: 53}
```

After applying, Pods can resolve DNS but cannot reach anything else.

---

## Lab

**Goal**: lock down a 3-tier mini-app — only the `web` Pods can reach `api`,
only `api` Pods can reach `db`; everything else is denied.

### Steps

1. Create namespace `l9-lab`.
2. Deploy 3 echo Pods labelled `tier=web`, `tier=api`, `tier=db` plus matching
   ClusterIP Services `web`, `api`, `db` on port 80.
3. Verify everything talks unrestricted (baseline).
4. Add a default-deny policy in the namespace (all ingress denied unless
   another policy allows).
5. Add a policy: `api` accepts traffic only from Pods with `tier=web`.
6. Add a policy: `db` accepts traffic only from Pods with `tier=api`.
7. Show that `web → api` works, `web → db` fails, `api → db` works.

### Solution

```bash
# 1
kubectl create ns l9-lab

# 2 — Each "tier" is a Pod-pair: an echo server + a busybox "tester" with wget,
#     both carrying the same `tier` label.
for t in web api db; do
  kubectl -n l9-lab run ${t}-svc --image=hashicorp/http-echo:1.0 \
    --labels=tier=$t,role=server --port=5678 \
    -- -listen=:5678 -text="hello from $t"
  kubectl -n l9-lab expose pod ${t}-svc --name=$t --port=80 --target-port=5678
  kubectl -n l9-lab run ${t}-tester --image=busybox:1.37 \
    --labels=tier=$t,role=tester \
    --command -- sh -c "sleep infinity"
done
kubectl -n l9-lab wait --for=condition=Ready pod -l tier --timeout=60s

# 3 — baseline reachability (everything works, no policy yet)
kubectl -n l9-lab exec web-tester -- wget -qO- -T 3 http://api/    # hello from api
kubectl -n l9-lab exec web-tester -- wget -qO- -T 3 http://db/     # hello from db
kubectl -n l9-lab exec api-tester -- wget -qO- -T 3 http://db/     # hello from db

# 4
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny, namespace: l9-lab}
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF

# 5
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: api-from-web, namespace: l9-lab}
spec:
  podSelector: {matchLabels: {tier: api}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {matchLabels: {tier: web}}
    ports: [{port: 5678, protocol: TCP}]
EOF

# 6
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: db-from-api, namespace: l9-lab}
spec:
  podSelector: {matchLabels: {tier: db}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {matchLabels: {tier: api}}
    ports: [{port: 5678, protocol: TCP}]
EOF
```

### Verification

```bash
# web -> api : allowed
kubectl -n l9-lab exec web-tester -- wget -qO- -T 5 http://api/
# Expected: "hello from api"

# web -> db  : blocked
kubectl -n l9-lab exec web-tester -- wget -qO- -T 5 http://db/
# Expected: download timed out (nonzero exit)

# api -> db  : allowed
kubectl -n l9-lab exec api-tester -- wget -qO- -T 5 http://db/
# Expected: "hello from db"
```

### Cleanup

```bash
kubectl delete ns l9-lab
```
