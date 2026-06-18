# Lesson 6 — Deploying the Cluster — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Take a prepared inventory, validate it, run `cluster.yml` to deploy a cluster,
retrieve the kubeconfig, and verify the cluster — the full Day-1 flow.

## Prerequisites

- Control node set up (`setup.md`): venv + `pip install -r requirements.txt`.
- A reachable fleet (Vagrant or 3 VMs) in `inventory/mycluster/inventory.ini`.

## Steps (énoncé)

1. Parse the inventory and confirm the groups are right.
2. `--syntax-check` `cluster.yml` before touching the nodes.
3. (🖥️) Deploy with `cluster.yml`.
4. Make the cluster reachable from the control node (kubeconfig).
5. Verify nodes and system Pods are healthy.

## Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoid the locale error
source ksenv/bin/activate && cd kubespray

# 1 — inventory (✅ validated output below)
ansible-inventory -i inventory/mycluster/inventory.ini --list | head

# 2 — syntax check (✅ validated: passes)
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml

# 3 — deploy (🖥️ needs the fleet; ~15-25 min)
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa playbooks/cluster.yml

# 4 — kubeconfig: either set kubeconfig_localhost: true in
#     group_vars/k8s_cluster/k8s-cluster.yml before deploy (artifacts/admin.conf),
#     or copy it from a control-plane node:
scp ubuntu@<cp1-ip>:/etc/kubernetes/admin.conf ~/.kube/config

# 5 — verify
kubectl get nodes
kubectl get pods -A
```

## Verification

**✅ Validated on the control node (no fleet needed):**

```text
$ ansible-inventory -i inventory/mycluster/inventory.ini --list
groups: ['all', 'etcd', 'k8s_cluster', 'kube_control_plane', 'kube_node']
hosts:  ['node1', 'node2', 'node3', 'node4', 'node5']

$ ansible-playbook ... --syntax-check playbooks/cluster.yml
playbook: playbooks/cluster.yml          # exit 0 — syntax OK
```

**🖥️ After deploy:** `kubectl get nodes` shows every node `Ready`; the CNI,
CoreDNS and kube-proxy Pods are `Running` in `kube-system`.

## Faster lab variant — all-in-one (single node)

Use a CI-validated single-node config as your group_vars base:

```bash
cp tests/files/ubuntu22-calico-all-in-one.yml \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
# point the inventory at one node in all three groups, then run cluster.yml
```

## Cleanup

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -b \
  -e reset_confirmation=yes playbooks/reset.yml
```
