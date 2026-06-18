# Lesson 4 — Cluster Configuration (group_vars) — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Turn a generic copied inventory into *your* cluster by editing the handful of
`group_vars` that shape every Kubespray cluster, then prove the edits are
syntactically valid and that the inventory still parses:

- `k8s_cluster/k8s-cluster.yml` — `kube_version`, `kube_network_plugin`,
  `kube_service_addresses`, `kube_pods_subnet`, `container_manager`.
- `k8s_cluster/addons.yml` — enable `metrics_server_enabled`, `helm_enabled`.
- `all/all.yml` — NTP, HTTP(S) proxy, and the API-server loadbalancer (optional).

Topology lives in the inventory (Lesson 2); **everything else lives in
`group_vars`** — that is what you edit here.

## Prerequisites

- Control node set up (`setup.md`): `python3 -m venv ksenv`, `git clone` of
  kubespray, `pip install -r requirements.txt`.
- A copied inventory from Lesson 2:
  `cp -rfp inventory/sample inventory/mycluster` — this brings the whole
  `group_vars/` tree (`all/`, `k8s_cluster/`) with inline-documented defaults.
- `export LC_ALL=C.UTF-8 LANG=C.UTF-8` to avoid the Ansible locale error.

The files you will edit (all real paths under `inventory/mycluster/`):

| File | Applies to | You set |
|------|-----------|---------|
| `group_vars/k8s_cluster/k8s-cluster.yml` | `k8s_cluster` | version, CNI, runtime, subnets |
| `group_vars/k8s_cluster/addons.yml` | `k8s_cluster` | metrics-server, helm |
| `group_vars/all/all.yml` | all hosts | NTP, proxy, loadbalancer |

## Steps (énoncé)

1. (Optional) Seed `k8s-cluster.yml` from a known-good CI scenario as a base
   (`tests/files/ubuntu22-calico-all-in-one.yml`), so you start from a config
   that already deploys, then adapt it.
2. Edit `group_vars/k8s_cluster/k8s-cluster.yml`: set `kube_version`,
   `kube_network_plugin: calico`, `container_manager: containerd`, and the
   `kube_service_addresses` / `kube_pods_subnet` CIDRs (non-overlapping).
3. Edit `group_vars/k8s_cluster/addons.yml`: enable `metrics_server_enabled`
   and `helm_enabled`.
4. Edit `group_vars/all/all.yml`: turn on NTP, (optionally) set an HTTP(S)
   proxy, and enable the internal API-server loadbalancer.
5. Validate each edited file is still valid YAML.
6. Re-run `ansible-inventory --list` to confirm the inventory still parses to
   the 5 groups with your `group_vars` applied (vars now show on the hosts).

## Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray

GV=inventory/mycluster/group_vars            # shorthand for the group_vars dir

# 1 — (optional) seed k8s-cluster.yml from a CI-validated scenario, then adapt
cp tests/files/ubuntu22-calico-all-in-one.yml $GV/k8s_cluster/k8s-cluster.yml
```

**2 — `group_vars/k8s_cluster/k8s-cluster.yml`** (the file that defines the
cluster). Set the key variables — edit in place, or ensure these lines exist
with your values:

```yaml
# inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
kube_version: v1.31.4              # Kubernetes release to deploy
kube_network_plugin: calico       # calico | cilium | flannel | kube-ovn | ...
container_manager: containerd     # containerd | crio | docker

# Service + Pod CIDRs — MUST NOT overlap each other or the node network
kube_service_addresses: 10.233.0.0/18
kube_pods_subnet: 10.233.64.0/18
kube_network_node_prefix: 24      # per-node pod subnet size

cluster_name: cluster.local
```

**3 — `group_vars/k8s_cluster/addons.yml`** (add-on toggles):

```yaml
# inventory/mycluster/group_vars/k8s_cluster/addons.yml
metrics_server_enabled: true      # kubectl top / HPA metrics
helm_enabled: true                # install the helm client on control plane
```

**4 — `group_vars/all/all.yml`** (cross-cutting, all hosts):

```yaml
# inventory/mycluster/group_vars/all/all.yml

# --- NTP: clock skew breaks TLS cert validation + etcd consensus ---
ntp_enabled: true
ntp_manage_config: true
ntp_servers:
  - "0.pool.ntp.org iburst"
  - "1.pool.ntp.org iburst"
ntp_timezone: Etc/UTC

# --- HTTP(S) proxy (optional; only if nodes reach the internet via a proxy) ---
# no_proxy is auto-generated to exclude all nodes + the LB — set only these two:
# http_proxy: "http://proxy.example.tld:3128"
# https_proxy: "http://proxy.example.tld:3128"

# --- Internal API-server loadbalancer (recommended for HA control plane) ---
loadbalancer_apiserver_localhost: true
loadbalancer_apiserver_type: nginx
```

Apply the edits with your editor of choice, e.g.:

```bash
vim $GV/k8s_cluster/k8s-cluster.yml
vim $GV/k8s_cluster/addons.yml
vim $GV/all/all.yml
```

```bash
# 5 — YAML validity of each edited file (✅ control node; prints OK or the error)
for f in $GV/k8s_cluster/k8s-cluster.yml \
         $GV/k8s_cluster/addons.yml \
         $GV/all/all.yml; do
  python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1])); print('OK', sys.argv[1])" "$f"
done

# 6 — inventory still parses, now with your group_vars applied (✅ control node)
ansible-inventory -i inventory/mycluster/inventory.ini --list | head -40
```

> **Precedence reminder:** `k8s_cluster/*` beats `all/*`, which beats Kubespray's
> role defaults; `host_vars` beats both; `-e` extra-vars always wins. Set the
> cluster-wide value here and override per host only when needed.

## Verification

**✅ Validated on the control node (no fleet, no deploy needed):**

```text
$ for f in .../k8s-cluster.yml .../addons.yml .../all.yml; do python3 -c "..."; done
OK inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
OK inventory/mycluster/group_vars/k8s_cluster/addons.yml
OK inventory/mycluster/group_vars/all/all.yml
# any indentation/quoting mistake fails here with a yaml.scanner error

$ ansible-inventory -i inventory/mycluster/inventory.ini --list
# still parses to: all / etcd / k8s_cluster / kube_control_plane / kube_node
# and the host entries now carry your vars, e.g.:
#   "kube_version": "v1.31.4",
#   "kube_network_plugin": "calico",
#   "metrics_server_enabled": true
```

Confirm a specific variable resolved as intended for the cluster nodes (✅):

```bash
ansible-inventory -i inventory/mycluster/inventory.ini --list \
  | python3 -c "import json,sys; d=json.load(sys.stdin); \
print(d['_meta']['hostvars']['node1'].get('kube_network_plugin'), \
      d['_meta']['hostvars']['node1'].get('kube_version'))"
# -> calico v1.31.4
```

**🖥️ Applied later, not in this lab:** these `group_vars` only take effect when
you run `cluster.yml` against the VM fleet in **Lesson 6**. Nothing here touches
a node.

## Cleanup

No cluster was deployed, so there is nothing to reset. To revert your config
edits back to Kubespray's documented defaults, restore the files from the
pristine sample:

```bash
# Restore the three edited files from inventory/sample (✅ control node)
cp inventory/sample/group_vars/k8s_cluster/k8s-cluster.yml \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
cp inventory/sample/group_vars/k8s_cluster/addons.yml \
   inventory/mycluster/group_vars/k8s_cluster/addons.yml
cp inventory/sample/group_vars/all/all.yml \
   inventory/mycluster/group_vars/all/all.yml

# ...or discard everything and re-copy the whole sample tree
rm -rf inventory/mycluster && cp -rfp inventory/sample inventory/mycluster
```
