# Lesson 17 — Upgrades — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Perform a **graceful, rolling upgrade** of a Kubespray-managed cluster the safe
way: snapshot etcd first, check out the matching Kubespray release tag, bump
`kube_version`, then run `playbooks/upgrade_cluster.yml` so nodes are
cordoned/drained/uncordoned **one at a time** — moving Kubernetes **one minor at
a time** (`1.30 → 1.31 → 1.32`, never skip).

## Prerequisites

- A running cluster deployed by **Lesson 6** (`cluster.yml`), reachable via the
  inventory at `inventory/mycluster/inventory.ini`.
- Working kubeconfig on the control node (`kubectl get nodes` returns `Ready`).
- The Kubespray git checkout you deployed with (so you can move it to the next
  tag). Know your **current** Kubespray tag and K8s minor before you start.

## Key facts (📖 before you touch anything)

- **`upgrade_cluster.yml` ≠ `cluster.yml`.** The upgrade playbook **cordons,
  drains, then uncordons** each node so workloads evacuate one batch at a time.
  `cluster.yml` does **not** drain.
- **Tag and version are coupled.** Each Kubespray tag targets a specific K8s
  range. Move Kubespray **one tag at a time**; let it move K8s **one minor at a
  time**. `v2.22.0 → v2.23.2 → v2.24.0` ✓ but `v2.22.0 → v2.24.0` ✗.
- **Never skip a minor** (patch jumps are fine): `1.30 → 1.31 → 1.32` ✓,
  `1.30 → 1.32` ✗. This respects kubeadm's version-skew rules.
- **No clean in-place rollback.** The safety net is a restorable **etcd
  snapshot**, taken *before* the upgrade.
- **Variable drift between releases.** group_vars formats/locations change
  between tags — diff the upstream sample inventory for the new tag first.

## Steps (énoncé)

1. (🖥️) Confirm the current cluster state and K8s version.
2. (🖥️) **Take an etcd snapshot first** — your only rollback path.
3. Move Kubespray to the **next** release tag (`git fetch --tags && git checkout`).
4. Reinstall pinned requirements for that tag; diff group_vars for drift.
5. Bump **`kube_version`** to the next minor in group_vars.
6. (✅) `--syntax-check` `playbooks/upgrade_cluster.yml` before touching nodes.
7. (🖥️) Run the graceful upgrade, **one node at a time** (`-e serial=1`).
8. (🖥️) Verify the node-by-node version progression, then repeat for the next minor.

## Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoid the locale error
source ksenv/bin/activate && cd kubespray

# 1 — current state (🖥️) — note the starting version, e.g. v1.30.x
kubectl get nodes -o wide
kubectl version

# 2 — etcd snapshot FIRST (🖥️) — your only rollback path.
#     Run on a control-plane node (paths from the kubeadm-managed etcd):
ssh ubuntu@<cp1-ip> 'sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  snapshot save /var/backups/etcd-pre-upgrade-$(date +%F).db'
# (Backup/restore is the focus of Lesson 11 — here it is the pre-flight gate.)

# 3 — move Kubespray ONE tag forward (📖→🖥️). Replace with the tag that
#     carries the next K8s minor (one tag at a time, never two):
git fetch --tags
git checkout v2.27.0          # example: the tag targeting your next K8s minor

# 4 — reinstall the tag's pinned deps, then check for variable drift:
pip install -r requirements.txt
diff -ruq inventory/sample/group_vars inventory/mycluster/group_vars | less
#   ^ reconcile any renamed/relocated vars before running (see drift note below).

# 5 — bump kube_version to the NEXT minor (one minor at a time):
#     inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
#     kube_version: v1.31.4        # was v1.30.x
# (or pass it on the CLI: -e kube_version=v1.31.4)
#
# ⚠️ Kubespray >= release-2.28 NORMALISED version strings: kube_version must
#    have **NO leading 'v'** (use `1.31.4`, not `v1.31.4`) — a validate_inventory
#    assert ("Stop if some versions have a 'v' left at the start") fails otherwise.
#    Also: upgrading to a NEW minor usually means upgrading **Kubespray itself**
#    first (e.g. release-2.27 caps at 1.31.x → checkout release-2.28 for 1.32.x,
#    then `pip install -r requirements.txt`), since the target version's binaries/
#    checksums must exist in the playbook.

# 6 — syntax check the upgrade playbook (✅ validated: passes)
ansible-playbook -i inventory/mycluster/inventory.ini \
  --syntax-check playbooks/upgrade_cluster.yml

# 7 — graceful upgrade, ONE node at a time (🖥️; safest). serial=1 overrides
#     the default of 20%-batches; cordon/drain/uncordon happens per node.
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa \
  -e kube_version=v1.31.4 -e serial=1 \
  playbooks/upgrade_cluster.yml

# 8 — verify, then REPEAT steps 3–7 for the next minor (1.31 → 1.32)
kubectl get nodes
kubectl version
```

### Controlling the roll-out (📖 — handy knobs)

```bash
# Refresh facts first when restricting to named nodes (avoids stale facts):
ansible-playbook -i inventory/mycluster/inventory.ini -b playbooks/facts.yml
ansible-playbook -i inventory/mycluster/inventory.ini -b \
  -e kube_version=v1.31.4 --limit "node4:node6:node7" \
  playbooks/upgrade_cluster.yml
```

| Control | Effect |
|---------|--------|
| `-e serial=1` | Upgrade **one node at a time** (default = batches of 20%). |
| `--limit "node4:node6"` | Restrict the upgrade to **specific nodes**. |
| `-e upgrade_node_confirm=true` | **Manual approval** before each node. |
| `-e upgrade_node_pause_seconds=60` | **Automatic pause** before each node. |
| `-e upgrade_node_post_upgrade_confirm=true` | Manual approval **after** each node. |

## Verification

**✅ Validated on the control node (no fleet needed):**

```text
$ ansible-playbook ... --syntax-check playbooks/upgrade_cluster.yml
playbook: playbooks/upgrade_cluster.yml      # exit 0 — syntax OK (ansible-core 2.18.x)
```

**🖥️ During/after the run — expected `kubectl get nodes` version progression.**
With `serial=1`, nodes flip to the new version **one at a time** (this is the
whole point — the cluster stays serviceable throughout):

```text
# before (start)
$ kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
node1   Ready    control-plane   20d   v1.30.4
node2   Ready    control-plane   20d   v1.30.4
node3   Ready    <none>          20d   v1.30.4

# mid-roll (node1 done, node2 cordoned/SchedulingDisabled, node3 not yet)
$ kubectl get nodes
NAME    STATUS                     ROLES           AGE   VERSION
node1   Ready                      control-plane   20d   v1.31.4
node2   Ready,SchedulingDisabled   control-plane   20d   v1.30.4
node3   Ready                      <none>          20d   v1.30.4

# after this minor completes — all on v1.31.4
$ kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
node1   Ready    control-plane   20d   v1.31.4
node2   Ready    control-plane   20d   v1.31.4
node3   Ready    <none>          20d   v1.31.4

$ kubectl version
Server Version: v1.31.4
```

Then **repeat** (new tag → bump `kube_version` to `v1.32.x` → run) to reach
`1.32`. Confirm `kubectl get nodes` is all-`Ready` and `kubectl get pods -A` is
healthy **between each minor** so any problem is caught and isolated early.

## CI-validated upgrade scenarios (📖 — known-good configs)

The upstream CI exercises the exact flow this lab follows (deploy an older
version, then `upgrade_cluster.yml` to a newer one). Use these as a config base
or reference:

```bash
# CRI-O runtime upgrade scenario
cat tests/files/ubuntu24-crio-upgrade.yml

# Calico CNI upgrade scenario
cat tests/files/debian11-calico-upgrade.yml
```

- `tests/files/ubuntu24-crio-upgrade.yml` — graceful upgrade with the CRI-O runtime.
- `tests/files/debian11-calico-upgrade.yml` — graceful upgrade with the Calico CNI.

> These `tests/files/*.yml` are complete, CI-validated configs. File names track
> the supported OS/release matrix and rotate over time — if a path 404s, browse
> `tests/files/` and pick the current `*-upgrade.yml` equivalent.

## Variable / inventory drift between versions (📖 — gotcha)

group_vars **formats and locations change between Kubespray tags**; an
out-of-date inventory copy will fail the upgrade. Always diff the upstream
sample for the new tag before running (step 4). Example relocation: from v2.19,
`etcd_kubeadm_enabled` is deprecated in favour of
`etcd_deployment_type: kubeadm`, moved to `group_vars/all/etcd.yml`.

## Pitfalls

- ❌ **Skipping a minor** (`1.30 → 1.32`) — unsupported; violates kubeadm
  version skew. Always go through `1.31`.
- ❌ **Jumping two Kubespray tags at once** — same rule applies to the tag.
- ❌ **No etcd snapshot** — there is no clean in-place rollback; the snapshot is
  your only recovery path.
- ❌ Using `cluster.yml -e upgrade_cluster_setup=true` for a K8s bump — migrates
  the control plane immediately but **bypasses graceful cordon/drain** →
  workload disruption. Prefer `upgrade_cluster.yml`.

## Cleanup

Nothing to tear down — this lab mutates a live cluster forward. To roll the lab
environment back to a clean baseline, restore the etcd snapshot from step 2
(Lesson 11) or redeploy from scratch:

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -b \
  -e reset_confirmation=yes playbooks/reset.yml      # full teardown
```
