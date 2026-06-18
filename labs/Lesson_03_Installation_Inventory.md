# Lesson 3 — Installation & Inventory — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Stand up the Kubespray **control node** and produce a parseable inventory:
clone Kubespray into a Python venv, install its Ansible deps, copy
`inventory/sample` → `inventory/mycluster`, **edit `inventory.ini`** to place
nodes in `kube_control_plane` / `etcd` / `kube_node` / `[k8s_cluster:children]`,
then prove the inventory parses (`ansible-inventory --list`) and — when a fleet
exists — that Ansible can reach every node (`ansible all -m ping -b`).

## Prerequisites

- A control node with **Python 3.11–3.13** and `git` (📖 see Lesson 1 / `setup.md`).
- Network access to clone from GitHub.
- (🖥️ only for the ping step) 3 reachable nodes with SSH + an account that can
  `sudo` to root, and your SSH key/IPs to hand. The inventory-parse steps below
  work **without** any of this.

## Steps (énoncé)

1. Create and activate a Python virtualenv, then clone Kubespray (pin a release
   tag — never run from a moving `master` in production).
2. `pip install -r requirements.txt` into the venv.
3. Copy the bundled sample inventory to your own `inventory/mycluster`.
4. **Edit `inventory.ini`**: declare your hosts, then place them in
   `[kube_control_plane]`, `[etcd]`, `[kube_node]`, and list the two child
   groups under `[k8s_cluster:children]`.
5. Parse the inventory and confirm the five groups appear
   (`ansible-inventory --list`).
6. (🖥️) Verify Ansible can SSH and become root on every node
   (`ansible all -m ping -b`).

> Note: the old `contrib/inventory_builder/inventory.py` script has been
> **removed** upstream — the inventory is edited **manually** now. ✅

## Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # ✅ avoids the locale error below

# 1 — venv + clone + pin a release tag  (✅)
python3 -m venv ksenv
source ksenv/bin/activate
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
git checkout release-2.27        # pin a tested tag; match it to your Python

# 2 — install Ansible deps into the venv  (✅)
pip install -U pip
pip install -r requirements.txt
ansible --version                # ansible-core 2.18.x

# 3 — copy the sample inventory (recursive, preserve attrs)  (✅)
cp -rfp inventory/sample inventory/mycluster

# 4 — edit the inventory by hand  (✅)
#     replace the placeholder nodeN lines with your hosts + IPs, then
#     assign each host to its group(s). Minimal 3-node example below.
${EDITOR:-vi} inventory/mycluster/inventory.ini
```

`inventory/mycluster/inventory.ini` — a worked 3-node example (1 schedulable
control-plane + etcd, 2 workers):

```ini
# --- hosts: ansible_host = SSH target; ip/access_ip = how k8s/peers reach it ---
[all]
node1 ansible_host=10.10.1.3 ip=10.10.1.3 access_ip=10.10.1.3
node2 ansible_host=10.10.1.4 ip=10.10.1.4 access_ip=10.10.1.4
node3 ansible_host=10.10.1.5 ip=10.10.1.5 access_ip=10.10.1.5

# --- control plane (apiserver, scheduler, controller-manager) ---
[kube_control_plane]
node1

# --- etcd: at least 3 in production; one here for a lab ---
[etcd]
node1

# --- workers: where Pods run ---
[kube_node]
node2
node3

# --- derived union — list the two child groups, never hand-edit hosts here ---
[k8s_cluster:children]
kube_control_plane
kube_node
```

> Per-host vars: `ansible_host` is the IP Ansible SSHes to; `ip` is the IP
> Kubernetes services bind to; `access_ip` is the IP other nodes use to reach
> this one. Set `ip`/`access_ip` only when they differ from `ansible_host`.
> A host listed in **both** `kube_control_plane` and `kube_node` is a
> **schedulable** control plane (fine for a lab; keep them disjoint for prod).

```bash
# 5 — parse the inventory (✅ validated output in Verification)
ansible-inventory -i inventory/mycluster/inventory.ini --list

# also handy:
ansible-inventory -i inventory/mycluster/inventory.ini --graph

# 6 — reachability check (🖥️ needs the fleet; -b = become root, required)
ansible -i inventory/mycluster/inventory.ini all -m ping -b \
  -u <ssh_user> --private-key ~/.ssh/id_rsa
```

## Verification

**✅ Validated on the control node (no fleet needed)** — with `ansible-core`
2.18.x, parsing the correctly-grouped `inventory.ini` yields exactly these five
groups:

```text
$ ansible-inventory -i inventory/mycluster/inventory.ini --list
# groups present:  all, etcd, k8s_cluster, kube_control_plane, kube_node

# abridged JSON:
{
  "all":                { "children": ["ungrouped", "k8s_cluster", "etcd"] },
  "k8s_cluster":        { "children": ["kube_control_plane", "kube_node"] },
  "kube_control_plane": { "hosts": ["node1"] },
  "etcd":               { "hosts": ["node1"] },
  "kube_node":          { "hosts": ["node2", "node3"] }
}
```

```text
$ ansible-inventory -i inventory/mycluster/inventory.ini --graph
@all:
  |--@k8s_cluster:
  |  |--@kube_control_plane:
  |  |  |--node1
  |  |--@kube_node:
  |  |  |--node2
  |  |  |--node3
  |--@etcd:
  |  |--node1
```

Confirm `k8s_cluster` is the **derived union** of `kube_control_plane` +
`kube_node` (you set it via `[k8s_cluster:children]`, never by listing hosts).

> If Ansible aborts with `could not initialize the preferred locale`, export
> `LC_ALL=C.UTF-8 LANG=C.UTF-8` (already in step 1) and re-run. ✅

**🖥️ After the ping step** (requires reachable nodes), each host reports
`SUCCESS` with `"ping": "pong"`:

```text
node1 | SUCCESS => { "changed": false, "ping": "pong" }
node2 | SUCCESS => { "changed": false, "ping": "pong" }
node3 | SUCCESS => { "changed": false, "ping": "pong" }
```

A `UNREACHABLE` line means SSH/key/user is wrong; a `become` failure means the
SSH user can't `sudo` to root — fix both before deploying (Lesson 6 needs `-b`).

## Cleanup

Nothing was changed on any target node (parse-only, plus a read-only ping), so
cleanup is just the control node:

```bash
deactivate                       # leave the venv
rm -rf inventory/mycluster       # drop your inventory copy (optional)
# to start completely over: rm -rf ~/kubespray ~/ksenv
```
