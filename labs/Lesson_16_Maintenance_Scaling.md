# Lesson 16 — Node Maintenance & Scaling — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Operate a **running** Kubespray cluster on Day-2 without disrupting its
workloads: **add a worker node** (append to the inventory under `kube_node`,
then `scale.yml --limit=<newnode>`), then **remove a node**
(`remove_node.yml -e node=<node>` — which cordons, drains and resets it — then
delete it from the inventory). You will also note the special procedures for
control-plane / etcd nodes and the **never-below-3 control-plane** rule.

## Prerequisites

- Control node set up (`setup.md`): venv + `pip install -r requirements.txt`,
  giving `ansible-core` 2.18.x.
- A **running** cluster from Lesson 6 with a working
  `inventory/mycluster/inventory.ini` and a kubeconfig
  (`kubectl get nodes` returns the existing nodes).
- One **new** host prepared (SSH reachable, key-based, sudo) to join as a
  worker — call it `node6`.

📖 Upstream reference: `docs/operations/nodes.md` ·
playbooks `playbooks/scale.yml`, `playbooks/remove_node.yml`, `playbooks/facts.yml`.

## Steps (énoncé)

**Part A — add a worker (`node6`):**

1. Note the current cluster state (`kubectl get nodes`).
2. Append `node6` to the inventory: add the host and put it **only** under
   `kube_node` (a worker is *not* in `etcd` or `kube_control_plane`).
3. Refresh the facts cache for **all** nodes first (`facts.yml`, no `--limit`) —
   so the limited scale run doesn't work from stale facts.
4. `--syntax-check` `scale.yml`.
5. (🖥️) Run `scale.yml` with `--limit=node6` to join only the new worker.
6. Verify `node6` shows up `Ready`.

**Part B — remove a node (`node5`):**

7. (🖥️) Run `remove_node.yml` with `-e node=node5` (cordons + drains + resets it)
   **while it is still in the inventory**.
8. Delete `node5` from the inventory file.
9. Verify it is gone from `kubectl get nodes`.

**Part C — note the special cases (📖, no run):**

10. Control-plane node → `cluster.yml`, never `scale.yml`; restart `nginx-proxy`.
11. etcd stays **odd**; you can't remove the *first* control-plane/etcd entry
    without reordering; never run an HA cluster **below 3** control-plane nodes.

## Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoid the locale error
source ksenv/bin/activate && cd kubespray
INV=inventory/mycluster/inventory.ini
```

### Part A — add a worker node

```bash
# 1 — current state (🖥️ on the live cluster)
kubectl get nodes

# 2 — append node6 to the inventory, in kube_node ONLY.
#     Edit inventory/mycluster/inventory.ini so it reads like:
#
#       [all]
#       ...
#       node6 ansible_host=10.0.0.16 ip=10.0.0.16
#
#       [kube_node]
#       node4
#       node5
#       node6            # <-- new worker, appended at the end
#
#     (do NOT add node6 to [etcd] or [kube_control_plane])

# confirm the inventory parses and node6 landed in kube_node (✅ no fleet needed)
ansible-inventory -i "$INV" --list | head
ansible-inventory -i "$INV" --graph kube_node

# 3 — refresh facts for ALL nodes BEFORE the limited run (no --limit here!)
#     (🖥️ touches every host; required so the --limit run isn't stale)
ansible-playbook -i "$INV" -b playbooks/facts.yml

# 4 — syntax check scale.yml (✅ validated: passes)
ansible-playbook -i "$INV" --syntax-check playbooks/scale.yml

# 5 — scale: join ONLY node6 (🖥️ needs the fleet; additive, ~5-10 min)
ansible-playbook -i "$INV" -u ubuntu -b --private-key ~/.ssh/id_rsa \
  --limit=node6 playbooks/scale.yml

# 6 — verify
kubectl get nodes
```

> `scale.yml` is the **additive worker path** — it joins new workers without
> re-running the whole cluster. Pair it with `--limit=node6` (and the prior
> `facts.yml`) to keep the blast radius on the new node only. Do **not** use
> `scale.yml` for a control-plane node.

### Part B — remove a node

```bash
# 7 — remove node5: cordon + drain + reset, WHILE it is still in the inventory
#     (🖥️ needs the fleet; the node is reset to a clean host by default)
ansible-playbook -i "$INV" -b -e node=node5 playbooks/remove_node.yml
#   Offline / unreachable node? add:
#     -e reset_nodes=false -e allow_ungraceful_removal=true
#   Removing several at once? -e node=node5,node6

# 8 — NOW delete node5 from inventory/mycluster/inventory.ini
#     (remove its line from [all] and from [kube_node])

# 9 — verify it's gone
kubectl get nodes
ansible-inventory -i "$INV" --graph kube_node     # ✅ node5 no longer listed
```

> **Order matters:** run `remove_node.yml` *first* (it needs the node to still be
> in the inventory to cordon/drain/reset it), *then* edit it out of the file.
> `remove_node.yml` is the single entry point for removing **any** node type —
> worker, control-plane or etcd.

### Part C — special cases (📖 reference, not run in this lab)

```bash
# Control-plane node: use cluster.yml (NOT scale.yml), append to the END of
# [kube_control_plane], then restart nginx-proxy on EVERY host so it reloads:
#   crictl ps | grep nginx-proxy | awk '{print $1}' | xargs crictl stop
ansible-playbook -i "$INV" -b playbooks/cluster.yml      # wires the new CP into certs/SANs

# etcd node: keep the count ODD (3, 5, …). Add via cluster.yml then
# upgrade-cluster.yml, both limited:
ansible-playbook -i "$INV" -b --limit=etcd,kube_control_plane \
  -e ignore_assert_errors=yes playbooks/cluster.yml
ansible-playbook -i "$INV" -b --limit=etcd,kube_control_plane \
  -e ignore_assert_errors=yes playbooks/upgrade_cluster.yml
```

- **Never below 3 control-plane / etcd members** in an HA cluster — scale back
  *up* before you remove any more. You **cannot remove the first** entry in
  `kube_control_plane` / `etcd` directly: reorder the inventory (push the first
  host to another position), re-apply with `cluster.yml` /
  `upgrade-cluster.yml`, *then* remove it; afterwards fix the `cluster-info`
  ConfigMap (`kubectl edit cm -n kube-public cluster-info`) to point at a live
  control-plane IP.

## Verification

**✅ Validated on the control node (no fleet needed):**

```text
$ ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/scale.yml
playbook: playbooks/scale.yml            # exit 0 — syntax OK

$ ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/remove_node.yml
playbook: playbooks/remove_node.yml      # exit 0 — syntax OK

$ ansible-inventory -i inventory/mycluster/inventory.ini --graph kube_node
@kube_node:
  |--node4
  |--node5
  |--node6                                # node6 appended as a worker
```

**🖥️ Expected `kubectl get nodes` before/after (needs the live cluster):**

```text
# --- before (3 CP/etcd + 2 workers) ---
$ kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
node1   Ready    control-plane   3d    v1.31.x
node2   Ready    control-plane   3d    v1.31.x
node3   Ready    control-plane   3d    v1.31.x
node4   Ready    <none>          3d    v1.31.x
node5   Ready    <none>          3d    v1.31.x

# --- after Part A (scale.yml --limit=node6): node6 joins Ready ---
$ kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
node1   Ready    control-plane   3d    v1.31.x
node2   Ready    control-plane   3d    v1.31.x
node3   Ready    control-plane   3d    v1.31.x
node4   Ready    <none>          3d    v1.31.x
node5   Ready    <none>          3d    v1.31.x
node6   Ready    <none>          1m    v1.31.x     # <-- added

# --- after Part B (remove_node.yml -e node=node5): node5 gone ---
$ kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
node1   Ready    control-plane   3d    v1.31.x
node2   Ready    control-plane   3d    v1.31.x
node3   Ready    control-plane   3d    v1.31.x
node4   Ready    <none>          3d    v1.31.x
node6   Ready    <none>          5m    v1.31.x     # node5 removed, node6 remains
```

The control plane stays at **3** throughout — only worker capacity changed.

## Cleanup

```bash
# Reverse this lab: remove node6 the same way you removed node5, then drop it
# from the inventory (🖥️).
ansible-playbook -i "$INV" -b -e node=node6 playbooks/remove_node.yml
# then delete node6 from [all] and [kube_node] in the inventory.

# Or tear the whole cluster down (Lesson 6 cleanup):
ansible-playbook -i "$INV" -b -e reset_confirmation=yes playbooks/reset.yml
```
