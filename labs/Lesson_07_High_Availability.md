# Lesson 7 — High Availability — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Build an **HA control plane**: put **3 nodes** in `kube_control_plane` **and**
`etcd`, choose how the kube-apiserver is load-balanced (localhost LB / external
`loadbalancer_apiserver` / `kube_vip` VIP), choose `etcd_deployment_type`
(`host` vs `kubeadm`) and **stacked vs external** etcd, deploy, then **prove HA**
by stopping one control-plane node and watching the API stay up.

## Prerequisites

- Control node set up (`setup.md`): venv + `pip install -r requirements.txt`,
  and `cp -rfp inventory/sample inventory/mycluster`.
- A fleet of **≥ 6 VMs** for the full HA test (3 control-plane/etcd + workers),
  or **≥ 7** for the separate-etcd variant. The all-in-one syntax/inventory steps
  (✅) need no nodes.
- `export LC_ALL=C.UTF-8 LANG=C.UTF-8` if Ansible hits the locale error.

## Steps (énoncé)

1. Edit the inventory: **3 nodes** in `kube_control_plane` with `etcd` stacked on
   them (odd count → quorum survives 1 failure), plus worker nodes.
2. Pick **one** API-server LB mode and set it in `group_vars/all/all.yml`:
   localhost LB (default), external `loadbalancer_apiserver`, or `kube_vip` VIP.
3. Pick `etcd_deployment_type` (`host` default, or `kubeadm` static-Pod) in
   `group_vars/all/etcd.yml`; for **external** etcd, move the members to their
   own nodes.
4. Parse the inventory and `--syntax-check` `cluster.yml` (✅).
5. (🖥️) Deploy with `cluster.yml`.
6. (🖥️) Verify HA: all control-plane Pods `Running`, `etcdctl member list` shows
   3 members, then **stop one control-plane node** and confirm the API survives.

## Solution

### 1 — HA inventory (✅ validated: parses to 3 CP / 3 etcd)

`inventory/mycluster/inventory.ini` — the **sample is already an HA layout**
(stacked etcd via `[etcd:children] kube_control_plane`); just uncomment 3 CP
nodes and the workers:

```ini
[kube_control_plane]
node1 ansible_host=10.10.0.11 ip=10.10.0.11 etcd_member_name=etcd1
node2 ansible_host=10.10.0.12 ip=10.10.0.12 etcd_member_name=etcd2
node3 ansible_host=10.10.0.13 ip=10.10.0.13 etcd_member_name=etcd3

# Stacked etcd: the etcd group == the control-plane nodes
[etcd:children]
kube_control_plane

[kube_node]
node4 ansible_host=10.10.0.14 ip=10.10.0.14
node5 ansible_host=10.10.0.15 ip=10.10.0.15
```

```bash
source ksenv/bin/activate && cd kubespray
ansible-inventory -i inventory/mycluster/inventory.ini --list | \
  python3 -c 'import json,sys;d=json.load(sys.stdin);print("etcd:",d["etcd"]["hosts"]);print("kube_control_plane:",d["kube_control_plane"]["hosts"])'
# ✅ etcd: ['node1','node2','node3']  kube_control_plane: ['node1','node2','node3']
```

### 2 — Choose the API-server LB mode — edit `group_vars/all/all.yml`

Pick **exactly one**. The variable names below are the real ones from the
sample `all.yml`.

**Option A — localhost loadbalancer (default, no extra infra).** Each non-master
node runs a local nginx proxy to the apiservers; nothing else to configure:

```yaml
loadbalancer_apiserver_localhost: true   # default = true
loadbalancer_apiserver_type: nginx       # or haproxy
loadbalancer_apiserver_port: 6443
loadbalancer_apiserver_healthcheck_port: 8081
```

**Option B — external load balancer (you provide the VIP).** Point every node at
one front-end VIP and wire its name into `/etc/hosts` + the API TLS SANs:

```yaml
apiserver_loadbalancer_domain_name: "apiserver-lb.lab.local"
loadbalancer_apiserver:
  address: 10.10.0.10
  port: 6443
loadbalancer_apiserver_localhost: false   # don't also run the per-node proxy
```

**Option C — kube-vip control-plane VIP (no external hardware).** kube-vip runs
as a static Pod on the control-plane and floats a VIP. ARP (layer-2) mode:

```yaml
kube_vip_enabled: true
kube_vip_controlplane_enabled: true
kube_vip_address: 10.10.0.42
kube_vip_arp_enabled: true
# kube_vip_interface: ens160     # interface that holds the VIP
kube_proxy_strict_arp: true      # required if kube_proxy_mode == ipvs
loadbalancer_apiserver:
  address: "{{ kube_vip_address }}"
  port: 6443
loadbalancer_apiserver_localhost: false
```

BGP mode instead of ARP (routed fabrics):

```yaml
kube_vip_enabled: true
kube_vip_controlplane_enabled: true
kube_vip_address: 10.10.0.42
kube_vip_bgp_enabled: true
kube_vip_local_as: 65000
kube_vip_bgp_routerid: 10.10.0.11
kube_vip_bgppeers:
  - 10.10.0.1:64512::false
loadbalancer_apiserver:
  address: "{{ kube_vip_address }}"
  port: 6443
loadbalancer_apiserver_localhost: false
```

📖 kube-vip reference: `docs/ingress/kube-vip.md` in the kubespray repo.

### 3 — Choose the etcd deployment type — edit `group_vars/all/etcd.yml`

```yaml
# host (default): etcd runs as a systemd service on the etcd nodes
etcd_deployment_type: host

# kubeadm (experimental, new clusters only): etcd as a static Pod on the
# control-plane hosts. Pair with skip_non_kubeadm_warning when not host-mode.
# etcd_deployment_type: kubeadm
# skip_non_kubeadm_warning: true
```

**Stacked** (above, the common default) keeps etcd on the control-plane nodes.
For **external etcd** put the members on their own nodes — separate the groups:

```ini
[kube_control_plane]
node1 ansible_host=10.10.0.11 ip=10.10.0.11
node2 ansible_host=10.10.0.12 ip=10.10.0.12
node3 ansible_host=10.10.0.13 ip=10.10.0.13

[etcd]
node6 ansible_host=10.10.0.16 ip=10.10.0.16 etcd_member_name=etcd1
node7 ansible_host=10.10.0.17 ip=10.10.0.17 etcd_member_name=etcd2
node8 ansible_host=10.10.0.18 ip=10.10.0.18 etcd_member_name=etcd3

[kube_node]
node4 ansible_host=10.10.0.14 ip=10.10.0.14
node5 ansible_host=10.10.0.15 ip=10.10.0.15
```

The CI scenario for this topology is `tests/files/ubuntu24-ha-separate-etcd.yml`
(3 × `kube_control_plane`, 3 × dedicated `etcd`, 1 × `kube_node`, Calico with
`calico_datastore: etcd`).

### 4 — Syntax check before touching nodes (✅ validated: passes)

```bash
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml
# ✅ playbook: playbooks/cluster.yml   (exit 0 — syntax OK)
```

### 5 — Deploy (🖥️ needs the fleet; ~20-30 min for HA)

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa playbooks/cluster.yml
```

### 6 — Fetch the kubeconfig and verify (🖥️)

```bash
# Point at the LB/VIP, not a single node:
scp ubuntu@node1:/etc/kubernetes/admin.conf ~/.kube/config
# If using Option B/C, edit the server: line to the VIP/domain name.
kubectl get nodes
```

## Verification

**✅ Validated on the control node (no fleet needed):**

```text
$ ansible-inventory -i inventory/mycluster/inventory.ini --list
etcd:               ['node1', 'node2', 'node3']
kube_control_plane: ['node1', 'node2', 'node3']      # 3 — odd, quorum tolerates 1 loss

$ ansible-playbook ... --syntax-check playbooks/cluster.yml
playbook: playbooks/cluster.yml          # exit 0 — syntax OK
```

**🖥️ After deploy — HA control plane is up:**

```bash
# Every control-plane node Ready, all control-plane Pods Running:
kubectl get nodes -l node-role.kubernetes.io/control-plane
kubectl -n kube-system get pods -o wide | \
  grep -E 'kube-apiserver|kube-controller-manager|kube-scheduler|etcd'
# expect 3 of each apiserver/controller-manager/scheduler, one per CP node
```

```bash
# etcd has 3 members (host deployment_type → etcdctl on an etcd node):
ssh ubuntu@node1 'sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-node1.pem \
  --key=/etc/ssl/etcd/ssl/admin-node1-key.pem'
# expect 3 rows, all started=true
# (kubeadm deployment_type: run etcdctl inside the etcd static Pod via crictl/kubectl)
```

**🖥️ Failover test — stop one control-plane node, API stays up:**

```bash
# Watch the API in one terminal:
while true; do kubectl get --raw='/readyz' && echo " @ $(date +%T)"; sleep 2; done

# In another, stop node1 (power off, or stop the kubelet + etcd):
ssh ubuntu@node1 'sudo systemctl stop kubelet etcd'   # or: virsh/multipass stop node1

# Expected:
#  - kubectl keeps returning "ok" (served by node2/node3 via the LB or VIP).
#  - etcd still has quorum (2 of 3) → cluster keeps accepting writes.
#  - after ~40s grace, node1 shows NotReady:
kubectl get nodes        # node1 NotReady, node2/node3 Ready
kubectl -n kube-system get pods -o wide | grep node1   # its static Pods gone/Pending

# Recover:
ssh ubuntu@node1 'sudo systemctl start etcd kubelet'
kubectl get nodes        # node1 back to Ready; etcd member list → 3 started again
```

If you used **kube-vip** (Option C), also confirm the VIP migrated: after
stopping the VIP holder, `ping 10.10.0.42` and `kubectl --server=https://10.10.0.42:6443`
keep working because kube-vip re-advertises the VIP from a surviving CP node.

## Faster lab variant — flannel HA scenario (CI-validated)

Use the upstream HA scenario as your group_vars base (3-node HA, `kubeadm` etcd):

```bash
cp tests/files/ubuntu24-flannel-ha.yml \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
# it sets: kube_network_plugin: flannel, etcd_deployment_type: kubeadm,
#          skip_non_kubeadm_warning: true
# keep the 3-CP/stacked-etcd inventory from step 1, then run cluster.yml
```

For the **external etcd** variant, base it on `tests/files/ubuntu24-ha-separate-etcd.yml`
and use the separated-groups inventory from step 3.

## Cleanup

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -b \
  -e reset_confirmation=yes playbooks/reset.yml
```
