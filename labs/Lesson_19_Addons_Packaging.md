# Lesson 19 — Add-ons & Packaging — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Turn on a set of Kubespray **add-ons** declaratively in one inventory file —
`metrics_server`, `helm`, `cert_manager`, `metallb` (with an address pool),
`local_volume_provisioner` and `registry` — re-run the playbook (full
`cluster.yml` or just `--tags=apps`), then **verify** each service: `kubectl top
nodes` (metrics-server), MetalLB `IPAddressPool`, and a real `type:
LoadBalancer` Service that gets an external IP.

> ☁️ **On the OpenStack lab cluster, skip MetalLB.** MetalLB is for **bare-metal**
> clusters with no cloud LB. Here the **OpenStack cloud-controller-manager +
> Octavia** already serve `type: LoadBalancer` Services (each gets a real
> **floating IP** — see L10 / setup.md); running both makes two controllers fight
> over the same Services. So enable `metrics_server_enabled`, `helm_enabled`,
> `cert_manager_enabled`, etc., but **leave `metallb_enabled: false`** and get LB
> IPs from Octavia. The MetalLB steps below are for a bare-metal target.
> ⚠️ A full `--tags=apps` re-run also re-applies the CCM DaemonSet — on Atmosphere
> that drops the live `hostAliases` DNS workaround, so re-patch it afterwards.

## Prerequisites

- A **deployed cluster** from Lesson 6 (`playbooks/cluster.yml`), reachable via
  `kubectl` (kubeconfig in place).
- Control node set up (`setup.md`): venv + `pip install -r requirements.txt`,
  inventory at `inventory/mycluster/inventory.ini`.
- Add-ons live in `inventory/mycluster/group_vars/k8s_cluster/addons.yml` (copied
  from `inventory/sample/` in Lesson 2).
- For MetalLB: a **free IP range on the node LAN** to hand out (here
  `192.168.121.240-192.168.121.250` — change to match your subnet).

## Steps (énoncé)

1. (✅) Enable the simple toggles in `addons.yml`: `metrics_server_enabled`,
   `helm_enabled`, `cert_manager_enabled`, `local_volume_provisioner_enabled`,
   `registry_enabled`.
2. (✅) Enable **MetalLB** and define an **address pool** in `metallb_config`;
   set the strict-ARP prerequisite.
3. (✅) Confirm `addons.yml` is **valid YAML** before running anything.
4. (🖥️) Apply the add-ons — re-run `cluster.yml`, or just `--tags=apps`.
5. (🖥️) Verify: Pods `Running`, `kubectl top nodes`, the `IPAddressPool`, and a
   `type: LoadBalancer` Service getting an IP.
6. (📖) Note on **removed ingress add-ons** → install with Helm post-deploy.

## Solution

### 1 — Simple add-on toggles (✅ edit `addons.yml`)

`inventory/mycluster/group_vars/k8s_cluster/addons.yml` — set:

```yaml
# --- Metrics & client tooling ---
metrics_server_enabled: true        # kubectl top + HPA
helm_enabled: true                  # Helm on control-plane (install charts post-deploy)

# --- Certificates ---
cert_manager_enabled: true          # cert-manager controller (issuers, auto-renew)

# --- Local static PVs (one PV per mount under host_dir) ---
local_volume_provisioner_enabled: true

# --- In-cluster private registry (runs in kube-system, Service on :5000) ---
registry_enabled: true
registry_namespace: kube-system
registry_storage_class: ""          # "" = emptyDir (lab); set a StorageClass for persistence
registry_disk_size: "10Gi"
```

> `local_volume_provisioner` is **static** (`kubernetes.io/no-provisioner`): it
> only exposes mounts that already exist under `host_dir`. For a real PV you must
> pre-mount a disk/tmpfs on the node and **persist it in `/etc/fstab`** — see the
> 🖥️ note at the end. Enabling the toggle alone deploys the provisioner cleanly
> even with no mounts yet.

### 2 — MetalLB + address pool (✅ edit `addons.yml`)

Same file, append:

```yaml
metallb_enabled: true
metallb_speaker_enabled: true
metallb_namespace: "metallb-system"

metallb_config:
  address_pools:
    primary:
      ip_range:
        - 192.168.121.240-192.168.121.250   # dash form; CIDR (e.g. 192.168.121.240/28) also works
      auto_assign: true
      avoid_buggy_ips: true                  # skip .0 and .255
  layer2:
    - primary                                # L2/ARP mode announces this pool
```

**Strict-ARP prerequisite** — MetalLB L2 needs kube-proxy strict ARP. Set it in
`inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`:

```yaml
kube_proxy_strict_arp: true
```

> Modes: `layer2:` (ARP, default — one node answers for the VIP, simplest) vs
> `layer3:` (BGP, peers under `metallb_peers`). For BGP-capable Calico ≥ 3.18 you
> can instead set `metallb_speaker_enabled: false` and use
> `calico_advertise_service_loadbalancer_ips`.

### 3 — Validate the YAML (✅ validated locally)

```bash
source ksenv/bin/activate && cd kubespray
python3 -c "import yaml; yaml.safe_load(open('inventory/mycluster/group_vars/k8s_cluster/addons.yml')); print('addons.yml: valid YAML')"
```

### 4 — Apply the add-ons (🖥️ needs the cluster)

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8

# Full converge (safe, idempotent) ...
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa playbooks/cluster.yml

# ... or just the add-ons stage (much faster on an existing cluster):
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa --tags=apps playbooks/cluster.yml
```

### 5 — Verify each add-on (🖥️)

```bash
# All add-on Pods up across namespaces
kubectl get pods -A

# metrics-server: node metrics must resolve (wait ~30-60s after deploy)
kubectl top nodes
kubectl top pods -A

# cert-manager controllers
kubectl get pods -n cert-manager

# in-cluster registry (kube-system) + its DaemonSet proxy on :5000
kubectl get pods,svc -n kube-system -l k8s-app=registry

# local volume provisioner (StorageClass uses no-provisioner)
kubectl get storageclass
kubectl get pv

# MetalLB: speaker/controller Pods + the address pool object
kubectl get pods -n metallb-system
kubectl get ipaddresspool -A
kubectl get l2advertisement -A
```

Now prove MetalLB hands out an external IP — deploy an app and a
`type: LoadBalancer` Service:

```bash
kubectl create deployment nginx --image=nginx --port=80
kubectl expose deployment nginx --type=LoadBalancer --port=80 --name=nginx-lb
kubectl get svc nginx-lb -w     # EXTERNAL-IP flips from <pending> to a pool IP
```

Equivalent manifest (✅ validated YAML):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

## Verification

**✅ Validated on the control node (no cluster needed):**

```text
$ python3 -c "import yaml; yaml.safe_load(open('.../addons.yml'))"
addons.yml: valid YAML          # toggles + metallb_config parse

# address pool round-trips to the expected shape:
metallb_config.address_pools.primary.ip_range = ['192.168.121.240-192.168.121.250']
layer2 = ['primary']

# the LoadBalancer Service manifest is valid: spec.type == LoadBalancer
```

**🖥️ After `cluster.yml` (or `--tags=apps`):**

- `kubectl get pods -A` — `metrics-server`, `cert-manager*`, `registry*`,
  `local-volume-provisioner*` and `metallb-system` (`controller` + `speaker`)
  Pods are `Running`.
- `kubectl top nodes` returns CPU/MEM per node (not an error) — metrics-server is
  serving.
- `kubectl get ipaddresspool -A` lists **`primary`** with range
  `192.168.121.240-192.168.121.250`.
- `kubectl get svc nginx-lb` shows an **`EXTERNAL-IP`** from that pool (no longer
  `<pending>`); `curl http://<external-ip>` returns the nginx welcome page.

## 📖 Note — removed ingress add-ons → install with Helm

Recent Kubespray releases **dropped some in-tree add-ons** (notably
**ingress-nginx** and the **Kubernetes Dashboard**). The supported path is to
install them **with Helm after the cluster is up** — which is exactly why
`helm_enabled: true` is in step 1. Example (🖥️, post-deploy, on a control-plane
node or any host with the kubeconfig):

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
# Its controller Service is type: LoadBalancer → MetalLB gives it a pool IP too.
```

Keep the **in-tree toggles** for cert-manager / registry / MetalLB / local PVs;
treat **nginx-ingress and dashboard as Helm charts**.

## 🖥️ Preparing a mount for the local volume provisioner (optional)

The provisioner only exposes mounts that already exist. On a `kube_node`, before
(or after) enabling it, create a mount under the class `host_dir` and **persist
it** so Pods don't break on reboot:

```bash
# Dev/tmpfs (ephemeral):
mkdir -p /mnt/disks/ssd1
mount -t tmpfs -o size=5G tmpfs /mnt/disks/ssd1

# Prod (real disk) — add to /etc/fstab so it survives reboot:
echo '/dev/vdb1  /mnt/disks/ssd1  ext4  defaults  0 0' >> /etc/fstab
```

After the next `--tags=apps` run, each mount appears as a `PV` from the
no-provisioner StorageClass (`kubectl get pv`).

## Cleanup

```bash
# Remove the demo LoadBalancer app
kubectl delete svc nginx-lb
kubectl delete deployment nginx

# To disable an add-on: flip its toggle back to false in addons.yml and re-run
#   ansible-playbook ... --tags=apps playbooks/cluster.yml
# (a full teardown is the Lesson 6 / setup.md reset)
ansible-playbook -i inventory/mycluster/inventory.ini -b \
  -e reset_confirmation=yes playbooks/reset.yml
```

---

## Concepts

| | **Helm** | **Kustomize** |
|--|----------|---------------|
| Approach | Package manager + **templating** | **Template-free** overlays |
| Unit | A **Chart** → a **Release** | Plain YAML + a `kustomization.yaml` |
| Tooling | Separate `helm` binary | **Built into `kubectl`** (`-k`) |
| Customize | `values.yaml`, `--set`, `-f` | bases/overlays, patches, generators |
| Lifecycle | install / upgrade / **rollback** / history | `apply -k` / `delete -k` |
| Best for | Off-the-shelf **components** & complex apps | **Your own** manifests per environment |

- **Helm 3** is **client-only** (no Tiller): it renders charts locally and submits
  the result to the API server, storing release state as **Secrets**.
- **Value precedence** (low → high): chart `values.yaml` < `-f` files (later wins)
  < **`--set`** (wins over everything).
- **Kustomize** has been in `kubectl` since **v1.14** — `kubectl apply -k` builds
  and applies; `kubectl kustomize` just renders.

---

## Examples — Helm

### 1. Scaffold a chart and render it (no cluster)

```bash
helm create demo
helm template demo ./demo | grep -E '^kind:|  name:|replicas:'
```

Sample output:

```
kind: ServiceAccount
  name: demo
kind: Service
  name: demo
kind: Deployment
  name: demo
  replicas: 1
```

> `helm template` is great for review/GitOps — it never touches the cluster.

### 2. Install a release, overriding a value

```bash
helm install demo ./demo -n helm-demo --create-namespace --set replicaCount=2
helm list -n helm-demo
kubectl -n helm-demo get deploy demo -o jsonpath='{.spec.replicas}{"\n"}'
```

Sample output:

```
NAME  NAMESPACE  REVISION  STATUS    CHART       APP VERSION
demo  helm-demo  1         deployed  demo-0.1.0  1.16.0
2
```

The deployment has **2** replicas — the `--set replicaCount=2` override beat the
chart default of 1.

### 3. Upgrade, inspect history, roll back

```bash
helm upgrade demo ./demo -n helm-demo --set replicaCount=4   # revision 2
helm history demo -n helm-demo
helm rollback demo 1 -n helm-demo                            # back to rev 1
kubectl -n helm-demo get deploy demo -o jsonpath='{.spec.replicas}{"\n"}'
```

Sample output:

```
REVISION  STATUS      CHART       DESCRIPTION
1         superseded  demo-0.1.0  Install complete
2         deployed    demo-0.1.0  Upgrade complete
...
2     # replicas after rollback to revision 1 (which had replicaCount=2)
```

- Upgrade to `replicaCount=4` → **4** replicas; rollback to rev 1 → back to **2**.
- Every change is a new **revision**; `helm rollback <name> <rev>` reverts to any.

### 4. Working with repositories (install off-the-shelf components)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm search repo ingress-nginx

# One command installs the whole component (Deployment, Service, RBAC, …):
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace --set controller.replicaCount=2
```

> `helm upgrade --install` is **idempotent** — installs if absent, upgrades if
> present. This is the canonical way to install cluster components with Helm.

### 5. Uninstall

```bash
helm uninstall demo -n helm-demo
```

---

## Examples — Kustomize

### 6. A base + a prod overlay

```bash
mkdir -p kz/base kz/overlays/prod
```

`kz/base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec:
  replicas: 1
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - {name: web, image: nginx:1.27, ports: [{containerPort: 80}]}
```

`kz/base/kustomization.yaml`:

```yaml
resources:
  - deployment.yaml
labels:
  - pairs: {app: web}
    includeSelectors: true
```

`kz/overlays/prod/replica-patch.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec: {replicas: 3}
```

`kz/overlays/prod/kustomization.yaml`:

```yaml
namespace: kz-prod
namePrefix: prod-
resources:
  - ../../base
patches:
  - path: replica-patch.yaml
images:
  - name: nginx
    newTag: 1.27-alpine
```

> Use `labels:` (with `pairs:`) — the old `commonLabels` is **deprecated**.

### 7. Render, then apply

```bash
kubectl kustomize kz/overlays/prod      # preview the built result
kubectl create ns kz-prod
kubectl apply -k kz/overlays/prod       # build + apply
kubectl -n kz-prod get deploy -o wide
```

Sample output:

```
# kubectl kustomize → name: prod-web, namespace: kz-prod, replicas: 3, image: nginx:1.27-alpine
deployment.apps/prod-web created
NAME       READY   UP-TO-DATE   AVAILABLE   CONTAINERS   IMAGES
prod-web   3/3     3            3           web          nginx:1.27-alpine
```

The overlay composed the base and applied **namePrefix** (`prod-`), a **namespace**
(`kz-prod`), a **patch** (replicas 1→3) and an **image** override — with **no
templating**.

---

## Lab

**Goal**: install the same app two ways — once with **Helm** (overriding a value),
once with **Kustomize** (a prod overlay over a base).

### Steps

1. **Helm:** scaffold a chart `site` (`helm create`), install it as release `site`
   in namespace `helm-lab` with **3 replicas** via `--set`. Upgrade it to image
   tag `1.27` (`--set image.tag=1.27`), then **roll back** to revision 1. Verify
   the replica count and revision history.
2. **Kustomize:** create a base (an nginx Deployment + Service) and a `staging`
   overlay that sets namespace `kz-staging`, a `stg-` name prefix, `replicas: 2`,
   and image tag `1.27-alpine`. Render with `kubectl kustomize`, then
   `kubectl apply -k`.

### Solution

```bash
# 1 — Helm
helm create site
helm install site ./site -n helm-lab --create-namespace --set replicaCount=3
helm upgrade site ./site -n helm-lab --set replicaCount=3 --set image.tag=1.27
helm history site -n helm-lab
helm rollback site 1 -n helm-lab
helm list -n helm-lab

# 2 — Kustomize
mkdir -p site-kz/base site-kz/overlays/staging
cat > site-kz/base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec:
  replicas: 1
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers: [{name: web, image: nginx:1.27, ports: [{containerPort: 80}]}]
EOF
cat > site-kz/base/kustomization.yaml <<'EOF'
resources: [deployment.yaml]
labels:
  - pairs: {app: web}
    includeSelectors: true
EOF
cat > site-kz/overlays/staging/replicas.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec: {replicas: 2}
EOF
cat > site-kz/overlays/staging/kustomization.yaml <<'EOF'
namespace: kz-staging
namePrefix: stg-
resources: [../../base]
patches: [{path: replicas.yaml}]
images: [{name: nginx, newTag: 1.27-alpine}]
EOF
kubectl kustomize site-kz/overlays/staging
kubectl create ns kz-staging
kubectl apply -k site-kz/overlays/staging
```

### Verification

```bash
# Helm: release present, deployment back to 3 replicas after rollback
helm list -n helm-lab                         # site, STATUS deployed
kubectl -n helm-lab get deploy site -o jsonpath='{.spec.replicas}{"\n"}'   # 3

# Kustomize: prefixed, namespaced, patched, image-overridden
kubectl -n kz-staging get deploy -o wide      # stg-web, 2/2, nginx:1.27-alpine
```

Expected: Helm release `site` deployed with the rolled-back spec (3 replicas);
Kustomize created `stg-web` in `kz-staging` with 2 replicas and the alpine image.

### Cleanup

```bash
helm uninstall site -n helm-lab
kubectl delete ns helm-lab kz-staging
rm -rf site-kz
```
