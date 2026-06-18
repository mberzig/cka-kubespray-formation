# Lesson 18 — Backup, Recovery & Reset — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Practice the Day-2 "protect and repair" flow: take a verified **etcd snapshot**
with `etcdctl`, **recover a broken control plane** from that snapshot with
`playbooks/recover_control_plane.yml`, and **tear the cluster down** cleanly with
`playbooks/reset.yml`. Finish by knowing the toggle for **Secret encryption at
rest**.

## Prerequisites

- Control node set up (`setup.md`): venv + `pip install -r requirements.txt`,
  `ansible-core` 2.18.x.
- An inventory at `inventory/mycluster/inventory.ini` (or `hosts.yaml`) with the
  `etcd` and `kube_control_plane` groups populated.
- (🖥️) A running cluster whose etcd uses the Kubespray default **`host`**
  deployment — etcd is a **systemd service** and the client TLS material lives
  under **`/etc/ssl/etcd/ssl/`**.
- SSH/`sudo` to at least one **etcd node** (for the snapshot) and **≥1 healthy
  node** (recovery needs one functional node).

## Steps (énoncé)

1. (📖) Identify the etcd client endpoint and the cert trio under
   `/etc/ssl/etcd/ssl/`.
2. (🖥️) On an etcd node, take a snapshot with `etcdctl snapshot save`.
3. (🖥️) Verify the snapshot with `etcdctl snapshot status --write-out=table`;
   copy it **off the node**.
4. ✅ `--syntax-check` `recover_control_plane.yml` and `reset.yml` before
   touching anything.
5. (📖) Prepare the recovery inventory: `broken_etcd` /
   `broken_kube_control_plane` groups, survivors listed first.
6. (🖥️) Run `recover_control_plane.yml` scoped to broken + healthy nodes, with
   `-e etcd_retries=...` and the snapshot path.
7. (🖥️) Tear down with `reset.yml -e reset_confirmation=yes`.
8. (📖) Note the encryption-at-rest toggle for later.

## Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoid the locale error
source ksenv/bin/activate && cd kubespray

# 4 — syntax checks (✅ validated: both pass on ansible-core 2.18.x)
ansible-playbook -i inventory/mycluster/inventory.ini \
  --syntax-check playbooks/recover_control_plane.yml
ansible-playbook -i inventory/mycluster/inventory.ini \
  --syntax-check playbooks/reset.yml
```

### 1–2 — Take an etcd snapshot (🖥️, run on an etcd node)

The `host` deployment exposes the etcd v3 client API on **`2379`**; the cert
trio is the CA plus this node's **admin** cert/key under `/etc/ssl/etcd/ssl/`.
Substitute `<host>` with this etcd node's hostname (matches the file names in
that directory).

```bash
# On the etcd node (e.g. node1). ETCDCTL_API=3 selects the v3 API (snapshots are v3-only).
sudo ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd_snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-<host>.pem \
  --key=/etc/ssl/etcd/ssl/admin-<host>-key.pem
```

### 3 — Verify the snapshot, then move it off-node (🖥️)

Never trust a backup you have not inspected — confirm the hash, revision and key
count:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd_snapshot.db --write-out=table

# Copy the snapshot off the node (to the control node / DR storage)
scp <user>@<etcd-node-ip>:/tmp/etcd_snapshot.db /tmp/etcd_snapshot.db
```

### 5 — Prepare the recovery inventory (📖)

In `inventory/mycluster/hosts.yaml` (or `inventory.ini`):

- Put broken **etcd** nodes in the **`broken_etcd`** group, each with
  `etcd_member_name` set.
- Put broken **control-plane** nodes in the **`broken_kube_control_plane`** group.
- List the **surviving** nodes **first** in `etcd` and `kube_control_plane`;
  add any **new/replacement** nodes **below** them.

### 6 — Recover the control plane (🖥️)

Scope `--limit` to the **broken member plus a healthy node** so the playbook can
rebuild the broken one from a working quorum. Raise `etcd_retries` (the right
value is hard to predict — go larger if it times out), and pass the **snapshot**
so a lost-quorum recovery restores from your known-good backup instead of the
first etcd node's auto-snapshot:

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml -b \
  --limit etcd,kube_control_plane \
  -e etcd_retries=10 \
  -e etcd_snapshot=/tmp/etcd_snapshot.db \
  playbooks/recover_control_plane.yml
```

> The playbook **detects whether etcd quorum is intact**. With quorum it rebuilds
> the broken members; with quorum **lost** it restores from a snapshot
> (`-e etcd_snapshot=...` selects yours). **Caveats (from the docs):** only
> tested with **small etcd DBs**, may cause **disruptions**, **no guarantees** —
> rehearse on a throwaway cluster first.

### 7 — Tear the cluster down (🖥️, destructive)

`reset.yml` is the inverse of `cluster.yml`: it wipes K8s components, etcd data,
CNI and config from the nodes. The `-e reset_confirmation=yes` flag skips the
interactive prompt — **back up etcd first** (steps 1–3).

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml -b \
  -e reset_confirmation=yes \
  playbooks/reset.yml
```

### 8 — Secret encryption at rest (📖, preventive toggle)

By default Kubernetes stores Secrets **base64-encoded, not encrypted**, in etcd —
so an etcd snapshot exposes them. Kubespray can wire an apiserver
`EncryptionConfiguration` to encrypt them at rest. In
`inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`:

```yaml
kube_encrypt_secret_data: true     # default false; provider defaults to secretbox
```

> `secretbox` is Kubespray's default provider (XSalsa20 + Poly1305). Enabling it
> only encrypts **newly written** Secrets until existing ones are re-written.
> See `docs/operations/encrypting-secret-data-at-rest.md`.

## Verification

**✅ Validated on the control node (no fleet needed):**

```text
$ ansible-playbook ... --syntax-check playbooks/recover_control_plane.yml
playbook: playbooks/recover_control_plane.yml      # exit 0 — syntax OK

$ ansible-playbook ... --syntax-check playbooks/reset.yml
playbook: playbooks/reset.yml                      # exit 0 — syntax OK
```

**🖥️ After `snapshot save`**, the command prints a success line and
`snapshot status --write-out=table` shows the snapshot's integrity hash,
revision, key count and size:

```text
Snapshot saved at /tmp/etcd_snapshot.db

+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 7d2c1f0a |   154832 |       1394 |     4.7 MB |
+----------+----------+------------+------------+
```

(Values vary per cluster — what matters is that the table prints with a non-zero
`TOTAL KEYS`/`TOTAL SIZE`; an error or empty table means a corrupt snapshot.)

**🖥️ After recovery:** the broken etcd member rejoins the cluster (verify with
`etcdctl member list --write-out=table` and `etcdctl endpoint health`), and
`kubectl get nodes` / `kubectl get pods -A` show the control plane healthy again.

**🖥️ After `reset.yml`:** the target nodes have no kubelet/etcd/CNI left;
`kubectl get nodes` from the old kubeconfig no longer reaches an apiserver. The
nodes are ready for a fresh `cluster.yml`.

## Faster lab variant — rehearse the recovery on a throwaway cluster

The docs explicitly recommend practicing first. Build a small all-in-one (or
2-etcd) cluster, snapshot it, deliberately break an etcd member, then run the
recovery against the snapshot:

```bash
# 1) snapshot a healthy etcd node (steps 1–3 above)
# 2) break a member on purpose (on that node):
sudo systemctl stop etcd && sudo rm -rf /var/lib/etcd/member
# 3) add it to broken_etcd (with etcd_member_name) in the inventory
# 4) run the recovery scoped to broken + a survivor:
ansible-playbook -i inventory/mycluster/hosts.yaml -b \
  --limit etcd,kube_control_plane \
  -e etcd_retries=10 -e etcd_snapshot=/tmp/etcd_snapshot.db \
  playbooks/recover_control_plane.yml
```

## Cleanup

```bash
# Tear the lab cluster down when finished (destructive):
ansible-playbook -i inventory/mycluster/inventory.ini -b \
  -e reset_confirmation=yes playbooks/reset.yml

# Remove local snapshot copies once no longer needed:
rm -f /tmp/etcd_snapshot.db
```
