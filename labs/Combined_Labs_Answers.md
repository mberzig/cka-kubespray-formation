# Kubernetes Production — CKA + Kubespray — Lab Answer Key (Instructor)

Reference solutions, expected verification output, and teaching notes for the 24 labs (Lab 0 setup + Labs 1–23) of the combined **Kubernetes Production — CKA + Kubespray** formation.

**All labs run on ONE shared cluster** — a multi-node Kubernetes cluster provisioned on **OpenStack VMs by Kubespray** (Lab 0). Lessons fall into two kinds:

- 🖥️ **Infra labs** run from the **Ansible control node** against `inventory/lab/` (inventory + `group_vars` edits, `cluster.yml`, `scale.yml`, `remove_node.yml`, `upgrade_cluster.yml`, `recover_control_plane.yml`, hardening, OpenStack integration). They need real multi-node VMs and **cannot run on kind**.
- **Usage labs** run with `kubectl` against the cluster's `admin.conf` — identical to any cluster; the OpenStack Octavia LB and Cinder CSI back the Access, Storage and Cloud labs.

Validation note: the **Kubespray/OpenStack steps are doc-validated** (cluster build, HA, scaling, upgrade, etcd recovery, hardening, cloud) — confirmed against the upstream docs and `ansible-playbook --syntax-check` / `ansible-inventory --list`, since they can't run on kind. The **usage-lab mechanics** (workloads, autoscaling, storage, access, RBAC, NetworkPolicy, CRDs, troubleshooting) were **validated on a local kind cluster** (1 control-plane + 2 workers, Kubernetes v1.31.2) and carry captured outputs (HPA `1 → 4 → 7`, etcd snapshot status table, ingress `v2`, etc.). Substitute your real node names (`kubectl get nodes`) for any `cka-lab-*` / `node1` examples.

## Summary

| Lab | Est. time | Difficulty |
|-----|-----------|------------|
| 0 — Deploy the cluster on OpenStack via Kubespray | 90 min | 🔴 🖥️ |
| 1 — Kubernetes Architecture | 10 min | 🟢 |
| 2 — Requirements & Environment | 20 min | 🟢 |
| 3 — Installation & Inventory | 25 min | 🟡 |
| 4 — Cluster Configuration (group_vars) | 30 min | 🟡 |
| 5 — Container Runtimes & CNI | 25 min | 🟡 |
| 6 — Deploying the Cluster (`cluster.yml`) | 45 min | 🔴 🖥️ |
| 7 — High Availability (failover) | 30 min | 🔴 🖥️ |
| 8 — Deploying Applications & Autoscaling | 25 min | 🟡 |
| 9 — Managing Storage (Cinder) | 15 min | 🟢 |
| 10 — Application Access (Octavia/Ingress/Gateway) | 25 min | 🟡 |
| 11 — Configuration & Quotas | 15 min | 🟢 |
| 12 — Scheduling | 20 min | 🟡 |
| 13 — Networking & NetworkPolicy | 20 min | 🟡 |
| 14 — Security (RBAC) | 20 min | 🟡 |
| 15 — CRDs & Operators | 15 min | 🟢 |
| 16 — Node Maintenance & Scaling | 35 min | 🔴 🖥️ |
| 17 — Upgrades (`upgrade-cluster.yml`) | 40 min | 🔴 🖥️ |
| 18 — Backup, Recovery & Reset | 35 min | 🔴 🖥️ |
| 19 — Add-ons & Packaging | 25 min | 🟡 |
| 20 — Troubleshooting | 20 min | 🟡 |
| 21 — Hardening & Offline | 20 min | 🟢 |
| 22 — Cloud & OpenStack | 25 min | 🟡 🖥️ |
| 23 — Practice Exam | 60 min | 🔴 |
| **TOTAL** | **690 min** | |

---

## Lab 0 — Deploy the cluster on OpenStack via Kubespray

**⏱ 90 min**   🔴   🖥️ *(Ansible control node)*

### Solution

```bash
# 1 — Ansible control node: clone Kubespray into a pinned venv
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
python3 -m venv venv && . venv/bin/activate
pip install -U pip && pip install -r requirements.txt

# 2 — Provision the VMs. Path A (recommended): Terraform builds VMs + dynamic inventory
cp -LRp contrib/terraform/openstack/sample-inventory inventory/lab
cd inventory/lab
ln -s ../../contrib/terraform/openstack/hosts        # dynamic inventory
ln -s ../../contrib/terraform/openstack              # tf modules
# edit cluster.tfvars: cluster_name, image, ssh_user, ext/internal network,
#   number_of_k8s_masters=3, number_of_k8s_nodes=2, flavors, floating IPs
source ~/openrc      # or export OS_* from application credentials
terraform -chdir=../../contrib/terraform/openstack init
terraform -chdir=../../contrib/terraform/openstack apply -var-file=$PWD/cluster.tfvars
cd ../..

# 3 — OpenStack cloud integration in group_vars/all/
#   all.yml:
#     cloud_provider: external
#     external_cloud_provider: openstack
#     cinder_csi_enabled: true
#     persistent_volumes_enabled: true
#   openstack.yml: auth_url, application_credential_id/secret, region,
#     external_openstack_lbaas_subnet_id, external_openstack_lbaas_floating_network_id

# 4 — Deploy (~20–40 min) and fetch the kubeconfig
ansible-playbook -i inventory/lab/hosts -b cluster.yml
export KUBECONFIG=$PWD/inventory/lab/artifacts/admin.conf   # kubeconfig_localhost=true default

# 5 — Verify the baseline every lab assumes
kubectl get nodes                       # 3 control-plane + 2 workers, all Ready
kubectl create deploy web --image=nginx:1.27
kubectl expose deploy web --port=80 --type=LoadBalancer
kubectl get svc web -w                  # EXTERNAL-IP becomes an Octavia floating IP
kubectl get sc                          # cinder-csi (cinder.csi.openstack.org)
kubectl delete deploy/web svc/web
```

A smaller variant (1 control-plane + 2 workers) is fine for everything except the HA failover lab (Lab 7), which needs 3 control planes.

### Verification

`kubectl get nodes` shows every node `Ready` (3 control-plane + 2 workers). The `type: LoadBalancer` Service receives a **floating IP from Octavia**, and `kubectl get sc` lists `cinder-csi` (`cinder.csi.openstack.org`) which provisions a real **Cinder** volume for any PVC. Keep `KUBECONFIG` pointed at `inventory/lab/artifacts/admin.conf` and the venv active — every later lab starts from there.

> **Teaching note:** This is the single shared cluster for the whole formation; everyone returns to it. Two prerequisites bite people: (1) the OpenStack tenant needs **Octavia (LBaaS)** *and* **Cinder** quota plus ≥5 floating IPs, or the Access/Storage labs have nothing to back them; (2) for L3 CNIs (Calico/kube-router) you must add **allowed-address-pairs** on the Neutron ports for the pod/service CIDRs (`10.233.0.0/18`, `10.233.64.0/18`) or OpenStack drops pod traffic as spoofed. Tear down with `reset.yml -e reset_confirmation=yes` (keeps the VMs) and `terraform destroy` (removes them).

---

## Lab 1 — Kubernetes Architecture

**⏱ 10 min**   🟢

### Solution

```bash
# 1 — node count + which is control plane (ROLES column)
kubectl get nodes

# 2 — the 4 control-plane Pods (static Pods on the control-plane node)
kubectl get pods -n kube-system \
  --field-selector spec.nodeName=$(kubectl get nodes -l node-role.kubernetes.io/control-plane= -o name | cut -d/ -f2) \
  -o name | grep -E 'etcd|apiserver|controller-manager|scheduler'

# 3 — a DaemonSet that runs on every node (and why)
kubectl get ds -n kube-system
# kube-proxy (Service routing) + the CNI (Calico/calico-node here) run on every node.

# 4 — API-server endpoint
kubectl cluster-info | grep -i 'control plane'

# 5 — where metrics-server runs (if installed)
kubectl get pod -n kube-system -l k8s-app=metrics-server -o wide
```

### Verification

Step 2 returns exactly four Pod names (`etcd-…`, `kube-apiserver-…`, `kube-controller-manager-…`, `kube-scheduler-…`). `kubectl get ds -n kube-system` shows `DESIRED == CURRENT` equal to the node count for `kube-proxy` and the CNI DaemonSet.

> **Teaching note:** Students look for control-plane components as Deployments — remind them these are *static Pods* managed by the kubelet from `/etc/kubernetes/manifests`, not by the scheduler. On this Kubespray cluster the CNI DaemonSet is `calico-node` (not `kindnet`), so the name they grep for depends on `kube_network_plugin`.

---

## Lab 2 — Requirements & Environment

**⏱ 20 min**   🟢

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoid the Ansible locale warning

# 1–2 (✅) control-node venv + Ansible exactly as Kubespray pins it
python3 -m venv ksenv && source ksenv/bin/activate
cd kubespray
pip install -U pip && pip install -r requirements.txt
ansible --version        # core within 2.18.x
python3 --version        # 3.11–3.13 for Ansible 2.18.x

# 3 (✅) netaddr present (Ansible's ipaddr filters need it)
python3 -c "import netaddr; print('netaddr', netaddr.__version__)"

# ---- 4–8 are 🖥️ : preflight the fleet before cluster.yml ----
# 4 reachability + become
ansible all -i inventory/mycluster/inventory.ini -m ping -b
# 5 supported OS
ansible all -i inventory/mycluster/inventory.ini -m setup -a 'filter=ansible_distribution*'
# 6 swap OFF (no output = good)
ansible all -i inventory/mycluster/inventory.ini -m shell -a 'swapon --show'
# 7 IPv4 forwarding (=1) + br_netfilter
ansible all -i inventory/mycluster/inventory.ini -m shell -a 'cat /proc/sys/net/ipv4/ip_forward'
ansible all -i inventory/mycluster/inventory.ini -m shell -a 'lsmod | grep br_netfilter || echo MISSING'
# 8 control-plane ports free pre-deploy (6443/2379/2380/10250)
ansible kube_control_plane -i inventory/mycluster/inventory.ini -b \
  -m shell -a "ss -tlnp | grep -E ':(6443|2379|2380|10250)' || echo 'none listening (expected pre-deploy)'"
```

### Verification

**✅ Control node:** `ansible --version` reports `core 2.18.x`; `import netaddr` prints `netaddr 1.x.x`.

**🖥️ Against the fleet:**

```text
node1 | SUCCESS => { "ping": "pong" }        # one line per node, all SUCCESS
$ ... swapon --show   → empty body            # swap OFF (good)
$ ... cat /proc/sys/net/ipv4/ip_forward → 1   # forwarding ON (good)
$ ... lsmod | grep br_netfilter → br_netfilter ... (or MISSING before deploy)
$ ... ss -tlnp ':(6443|2379|2380|10250)' → none listening (expected pre-deploy)
```

After `cluster.yml` the same `ss` command shows `kube-apiserver:6443`, `etcd:2379/2380`, `kubelet:10250` all `LISTEN`.

> **Teaching note:** Kubespray's `preinstall` role loads `br_netfilter`/`overlay`, sets the bridge sysctls and turns swap off — so steps 6–7 are a *preflight* to catch a locked-down image *before* the deploy, not a substitute for the role. A `UNREACHABLE` ping means SSH/key/user is wrong; a `become` failure means the SSH user can't `sudo` to root — fix both before Lab 6, which needs `-b`.

---

## Lab 3 — Installation & Inventory

**⏱ 25 min**   🟡

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoids the locale error

# 1 (✅) venv + clone + pin a release tag (never run from a moving master)
python3 -m venv ksenv && source ksenv/bin/activate
git clone https://github.com/kubernetes-sigs/kubespray.git && cd kubespray
git checkout release-2.27        # pin a tested tag; match it to your Python

# 2 (✅) install deps
pip install -U pip && pip install -r requirements.txt
ansible --version                # ansible-core 2.18.x

# 3 (✅) copy the sample inventory (recursive, preserve attrs)
cp -rfp inventory/sample inventory/mycluster

# 4 (✅) edit the inventory by hand (the old inventory_builder script is GONE)
${EDITOR:-vi} inventory/mycluster/inventory.ini
```

`inventory/mycluster/inventory.ini` — a worked 3-node example:

```ini
[all]
node1 ansible_host=10.10.1.3 ip=10.10.1.3 access_ip=10.10.1.3
node2 ansible_host=10.10.1.4 ip=10.10.1.4 access_ip=10.10.1.4
node3 ansible_host=10.10.1.5 ip=10.10.1.5 access_ip=10.10.1.5

[kube_control_plane]
node1
[etcd]
node1
[kube_node]
node2
node3
[k8s_cluster:children]
kube_control_plane
kube_node
```

```bash
# 5 (✅) parse the inventory; confirm the five groups appear
ansible-inventory -i inventory/mycluster/inventory.ini --list
ansible-inventory -i inventory/mycluster/inventory.ini --graph

# 6 (🖥️) reachability (-b = become root, required)
ansible -i inventory/mycluster/inventory.ini all -m ping -b -u <ssh_user> --private-key ~/.ssh/id_rsa
```

### Verification

**✅ Control node** — parsing yields exactly five groups and the derived union:

```text
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

`k8s_cluster` is the union of `kube_control_plane` + `kube_node` (set via `[k8s_cluster:children]`, never by listing hosts).

**🖥️ ping step:** every host reports `SUCCESS => { "ping": "pong" }`.

> **Teaching note:** Two recurring failures: (1) the upstream `contrib/inventory_builder/inventory.py` is **removed** — the inventory is edited by hand now, so don't teach the old script. (2) If Ansible aborts with `could not initialize the preferred locale`, export `LC_ALL=C.UTF-8 LANG=C.UTF-8`. Clarify the per-host vars: `ansible_host` = the SSH target, `ip` = the IP Kubernetes binds to, `access_ip` = how other nodes reach this one — set the latter two only when they differ.

---

## Lab 4 — Cluster Configuration (group_vars)

**⏱ 30 min**   🟡

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray
GV=inventory/mycluster/group_vars

# 1 (optional) seed k8s-cluster.yml from a CI-validated scenario, then adapt
cp tests/files/ubuntu22-calico-all-in-one.yml $GV/k8s_cluster/k8s-cluster.yml
```

**2 — `group_vars/k8s_cluster/k8s-cluster.yml`** (defines the cluster):

```yaml
kube_version: v1.31.4
kube_network_plugin: calico
container_manager: containerd
kube_service_addresses: 10.233.0.0/18     # MUST NOT overlap pods or node net
kube_pods_subnet: 10.233.64.0/18
kube_network_node_prefix: 24
cluster_name: cluster.local
```

**3 — `group_vars/k8s_cluster/addons.yml`:** `metrics_server_enabled: true`, `helm_enabled: true`.

**4 — `group_vars/all/all.yml`:** NTP on (`ntp_enabled`/`ntp_servers`/`ntp_timezone`), optional HTTP(S) proxy, and the internal API-server LB (`loadbalancer_apiserver_localhost: true`, `loadbalancer_apiserver_type: nginx`).

```bash
# 5 (✅) YAML validity of each edited file
for f in $GV/k8s_cluster/k8s-cluster.yml $GV/k8s_cluster/addons.yml $GV/all/all.yml; do
  python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1])); print('OK', sys.argv[1])" "$f"
done

# 6 (✅) inventory still parses, now with your group_vars applied
ansible-inventory -i inventory/mycluster/inventory.ini --list | head -40
```

### Verification

**✅ Control node:** each file prints `OK …` (a quoting/indent mistake fails here with a `yaml.scanner` error). The inventory still parses to the 5 groups and the host vars now carry your values:

```bash
ansible-inventory -i inventory/mycluster/inventory.ini --list \
  | python3 -c "import json,sys; d=json.load(sys.stdin); h=d['_meta']['hostvars']['node1']; \
print(h.get('kube_network_plugin'), h.get('kube_version'))"
# -> calico v1.31.4
```

These `group_vars` only take effect when `cluster.yml` runs against the fleet (Lab 6) — nothing here touches a node.

> **Teaching note:** Drill the **precedence**: `k8s_cluster/*` beats `all/*`, which beats Kubespray's role defaults; `host_vars` beats both; `-e` extra-vars always wins. Topology lives in the *inventory*; *everything else* lives in `group_vars`. Service and Pod CIDRs must not overlap each other or the node network — the #1 silent misconfiguration here.

---

## Lab 5 — Container Runtimes & CNI

**⏱ 25 min**   🟡

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray

# 1 (✅) container_manager is the ONE runtime switch (lives in k8s-cluster.yml)
grep -n container_manager inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
# container_manager: containerd   # default; set to 'crio' for CRI-O
```

**2 — containerd tuning** in `group_vars/all/containerd.yml`: `SystemdCgroup: "true"` on the runc runtime, and `containerd_registries_mirrors` (a matched `prefix` + a list of `mirrors`; an insecure HTTP registry is a mirror with `http://` + `skip_verify: true`).

```bash
# 3 (✅) YAML validity
python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1])); print('YAML OK')" \
  inventory/mycluster/group_vars/all/containerd.yml
ansible-inventory -i inventory/mycluster/inventory.ini --list >/dev/null && echo "inventory OK"

# 4 (✅) ALTERNATIVE — CRI-O from a CI scenario
cp tests/files/almalinux9-crio.yml inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
#   + container_manager: crio in all.yml ; registries in group_vars/all/cri-o.yml

# --- CNI: pick with the single kube_network_plugin switch ---
GV=inventory/mycluster/group_vars/k8s_cluster
cp tests/files/almalinux9-calico.yml $GV/k8s-cluster.yml   # Calico is the default
# CIDRs (non-overlapping):
#   kube_service_addresses: 10.233.0.0/18 ; kube_pods_subnet: 10.233.64.0/18 ; node_prefix 24
# Calico encap in k8s-net-calico.yml: calico_vxlan_mode: CrossSubnet ; calico_ipip_mode: Never
# Connectivity self-test: deploy_netchecker: true

# 5 (✅) validate + syntax-check the playbook
python3 -c "import yaml,sys; [yaml.safe_load(open(f)) for f in sys.argv[1:]]; print('YAML OK')" \
  $GV/k8s-cluster.yml $GV/k8s-net-calico.yml
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml
```

**🖥️ After deploy:** verify the live runtime with `sudo crictl info | grep -iE 'RuntimeName|SystemdCgroup'`, list workloads with `crictl ps`/`crictl pods`; optionally enable `gvisor_enabled: true` and run a Pod with `runtimeClassName: gvisor`. For the CNI, check `calico-node` Pods, query netchecker (`/api/v1/connectivity_check`), and ping cross-node (`p1 → p2`).

### Verification

**✅ Control node:** `YAML OK` for the containerd/cri-o/calico files; `inventory OK`; `--syntax-check` exits 0. The grep confirms `kube_network_plugin: calico`, the two CIDRs, `deploy_netchecker: true`, and `calico_vxlan_mode: CrossSubnet` / `calico_ipip_mode: Never`.

**🖥️ After deploy:** `crictl info` reports the expected `RuntimeName` and `SystemdCgroup: true`; the CNI Pods (`calico-node`) are `Running` on every node; netchecker reports `"Connectivity to all nodes is OK"`; the cross-node `ping p1 → p2` shows `0% packet loss`; with gVisor, the `gvisor-test` Pod is `Running` and `dmesg`/`uname` inside it report the sandbox kernel, not the host's.

> **Teaching note:** gVisor and Kata are **not** `container_manager` values — they ride on top of containerd and are selected **per-Pod** via `RuntimeClass`. The cgroup driver (`SystemdCgroup`) must match the kubelet's, or kubelets fail to start. `calico_vxlan_mode`/`calico_ipip_mode` only apply when `kube_network_plugin: calico`; Cilium/Flannel have their own tunables.

---

## Lab 6 — Deploying the Cluster (`cluster.yml`)

**⏱ 45 min**   🔴   🖥️ *(Ansible control node)*

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray

# 1 (✅) parse the inventory, confirm groups
ansible-inventory -i inventory/mycluster/inventory.ini --list | head

# 2 (✅) syntax-check BEFORE touching nodes
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml

# 3 (🖥️) deploy (~15–25 min)
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa playbooks/cluster.yml

# 4 kubeconfig: set kubeconfig_localhost: true before deploy (artifacts/admin.conf),
#   or copy it from a control-plane node:
scp ubuntu@<cp1-ip>:/etc/kubernetes/admin.conf ~/.kube/config

# 5 verify
kubectl get nodes
kubectl get pods -A
```

### Verification

**✅ Control node:** the inventory lists `['all','etcd','k8s_cluster','kube_control_plane','kube_node']`; `--syntax-check` prints `playbook: playbooks/cluster.yml` and exits 0.

**🖥️ After deploy:** `kubectl get nodes` shows every node `Ready`; the CNI, CoreDNS and kube-proxy Pods are `Running` in `kube-system`.

> **Teaching note:** Always `--syntax-check` first — it catches inventory/group_vars mistakes in seconds without SSHing anywhere. For a quick single-node smoke test, base `k8s-cluster.yml` on `tests/files/ubuntu22-calico-all-in-one.yml` and put one host in all three groups. `cluster.yml` is **idempotent** — re-running it converges rather than rebuilding, which is how add-ons (Lab 19) and config changes are applied later.

---

## Lab 7 — High Availability (failover)

**⏱ 30 min**   🔴   🖥️ *(Ansible control node + fleet)*

### Solution

```bash
# 1 (✅) HA inventory: 3 nodes in kube_control_plane WITH stacked etcd (odd → tolerates 1 loss)
#   The sample is already an HA layout via [etcd:children] kube_control_plane:
#     [kube_control_plane] node1 node2 node3   (each with etcd_member_name=etcdN)
#     [etcd:children] kube_control_plane
#     [kube_node] node4 node5
ansible-inventory -i inventory/mycluster/inventory.ini --list | \
  python3 -c 'import json,sys;d=json.load(sys.stdin);print("etcd:",d["etcd"]["hosts"]);print("cp:",d["kube_control_plane"]["hosts"])'

# 2 — pick ONE API-server LB mode in group_vars/all/all.yml:
#   A) localhost LB (default): loadbalancer_apiserver_localhost: true ; type: nginx
#   B) external LB/VIP: apiserver_loadbalancer_domain_name + loadbalancer_apiserver{address,port}
#   C) kube-vip: kube_vip_enabled + kube_vip_controlplane_enabled + kube_vip_address + ARP/BGP

# 3 — etcd deployment type in group_vars/all/etcd.yml:
#   etcd_deployment_type: host   (default; systemd service) | kubeadm (static Pod)
#   STACKED = etcd on the CP nodes (default) ; EXTERNAL = move members to their own nodes

# 4 (✅) syntax check
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml

# 5 (🖥️) deploy (~20–30 min for HA)
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa playbooks/cluster.yml
```

**6 — verify HA + failover (🖥️):**

```bash
# all 3 control-plane Pods + 3 etcd members
kubectl -n kube-system get pods -o wide | grep -E 'kube-apiserver|controller-manager|scheduler|etcd'
ssh ubuntu@node1 'sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-node1.pem --key=/etc/ssl/etcd/ssl/admin-node1-key.pem'

# failover: watch the API, then stop one control-plane node
while true; do kubectl get --raw='/readyz' && echo " @ $(date +%T)"; sleep 2; done
ssh ubuntu@node1 'sudo systemctl stop kubelet etcd'
kubectl get nodes            # node1 NotReady after ~40s; node2/node3 Ready
ssh ubuntu@node1 'sudo systemctl start etcd kubelet'   # recover → back to Ready, 3 etcd members
```

### Verification

**✅ Control node:** the inventory shows `etcd` and `kube_control_plane` each = `['node1','node2','node3']` (3 — odd, tolerates 1 loss); `--syntax-check` exits 0.

**🖥️ After deploy + failover:** all 3 control-plane Pods `Running` (one per CP node); `etcdctl member list` shows 3 rows `started=true`. With one CP node stopped, `kubectl get --raw=/readyz` keeps returning `ok` (served via the LB/VIP) and etcd keeps quorum (2 of 3); after ~40s the stopped node shows `NotReady`. After `systemctl start`, it returns to `Ready` and etcd is back to 3 members. With kube-vip, the VIP migrates to a surviving CP node.

> **Teaching note:** Two exam-critical facts: keep an **odd etcd quorum** (3 tolerates 1 failure) and set the API-server LB mode **before** deploy. Stacked vs external etcd is a topology choice — stacked is simpler (members on the CP nodes), external isolates etcd onto dedicated nodes (`tests/files/ubuntu24-ha-separate-etcd.yml`). The failover test proves the LB, not just etcd — point students at `/readyz` staying `ok` while a master is down.

---

## Lab 8 — Deploying Applications & Autoscaling

**⏱ 25 min**   🟡

### Solution

```bash
# --- Part 1: Deployment / DaemonSet / StatefulSet ---
kubectl create ns l3-lab

# Deployment web, label propagated to the POD TEMPLATE too
kubectl -n l3-lab create deployment web --image=nginx:1.27 --replicas=3
kubectl -n l3-lab label deployment web tier=frontend
kubectl -n l3-lab patch deployment web --type=merge \
  -p '{"spec":{"template":{"metadata":{"labels":{"tier":"frontend"}}}}}'

kubectl -n l3-lab set image deployment/web nginx=nginx:1.28
kubectl -n l3-lab rollout status deployment/web
kubectl -n l3-lab scale deployment/web --replicas=5
kubectl -n l3-lab scale deployment/web --replicas=2
kubectl -n l3-lab rollout undo deployment/web --to-revision=1   # back to nginx:1.27

# DaemonSet (one busybox per worker)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata: {name: agent, namespace: l3-lab}
spec:
  selector: {matchLabels: {app: agent}}
  template:
    metadata: {labels: {app: agent}}
    spec:
      containers: [{name: agent, image: busybox:1.37, command: ["sh","-c","sleep infinity"]}]
EOF

# Headless Service + StatefulSet (50Mi per-replica PVC)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata: {name: data, namespace: l3-lab}
spec: {clusterIP: None, selector: {app: data}, ports: [{port: 80, name: data}]}
---
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: data, namespace: l3-lab}
spec:
  serviceName: data
  replicas: 2
  selector: {matchLabels: {app: data}}
  template:
    metadata: {labels: {app: data}}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        volumeMounts: [{name: data, mountPath: /usr/share/nginx/html}]
  volumeClaimTemplates:
  - metadata: {name: data}
    spec: {accessModes: ["ReadWriteOnce"], resources: {requests: {storage: 50Mi}}}
EOF

# --- Part 2: Autoscaling (HPA) ---
kubectl create ns hpa-lab
kubectl -n hpa-lab create deployment web --image=registry.k8s.io/hpa-example
kubectl -n hpa-lab set resources deployment web --requests=cpu=100m --limits=cpu=500m
kubectl -n hpa-lab expose deployment web --port=80
kubectl -n hpa-lab wait --for=condition=Available deployment/web --timeout=300s

cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: web, namespace: hpa-lab}
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: web}
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource: {name: cpu, target: {type: Utilization, averageUtilization: 50}}
EOF

kubectl -n hpa-lab run load --image=busybox:1.37 --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://web >/dev/null; done"
kubectl -n hpa-lab get hpa web -w
kubectl -n hpa-lab delete pod load    # scale back in after the ~5-min window
```

### Verification

Part 1: `kubectl -n l3-lab get deploy,ds,sts` — deployment back to `nginx:1.27` after the rollback; 2 DaemonSet Pods (one per worker); 2 StatefulSet Pods `data-0`/`data-1` with PVCs `data-data-0`/`data-data-1`.

Part 2 — captured HPA run (min=1 demo):

```text
TARGETS              REPLICAS
cpu: <unknown>/50%   1
cpu: 250%/50%        1        # load hits
cpu: 164%/50%        4        # scaled up
cpu: 84%/50%         7
cpu: 36%/50%         7        # stabilised under target at 7
```

The HPA scaled `1 → 4 → 7` as CPU spiked to 250%, then held at 7. With min=2/max=8 it pushes toward 8; after `delete pod load`, replicas trend back toward 2.

> **Teaching note:** Labeling the Deployment object does **not** label the Pod template — students forget the `patch` on `spec.template.metadata.labels`, so `get pod -l tier=frontend` returns nothing. For HPA the #1 failure is `TARGETS: <unknown>`: caused by no metrics-server or Pods with no `requests.cpu` (Utilization is a % of the request). Scale-**down** is deliberately slow (`--horizontal-pod-autoscaler-downscale-stabilization`, default 5 min). HPA scales **Pods**; VPA tunes requests; Cluster Autoscaler scales **nodes**.

---

## Lab 9 — Managing Storage (Cinder)

**⏱ 15 min**   🟢

### Solution

```bash
kubectl create ns l4-lab

# 200Mi dynamic PVC (default SC — cinder-csi on this cluster)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: app-data, namespace: l4-lab}
spec: {accessModes: ["ReadWriteOnce"], resources: {requests: {storage: 200Mi}}}
EOF

# writer writes Hello-$RANDOM into /data/marker
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: writer, namespace: l4-lab}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","echo Hello-$RANDOM > /data/marker && tail -f /dev/null"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: app-data}}]
EOF
kubectl -n l4-lab wait --for=condition=Ready pod/writer --timeout=60s
MARKER=$(kubectl -n l4-lab exec writer -- cat /data/marker); echo "Wrote: $MARKER"

# delete writer; a NEW reader Pod on the same PVC must see the same value
kubectl -n l4-lab delete pod writer
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: reader, namespace: l4-lab}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh","-c","cat /data/marker; tail -f /dev/null"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: app-data}}]
EOF
kubectl -n l4-lab wait --for=condition=Ready pod/reader --timeout=60s
kubectl -n l4-lab exec reader -- cat /data/marker   # must equal $MARKER

# static PV + PVC bound by a shared storageClassName: archive
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata: {name: archive-pv}
spec:
  capacity: {storage: 100Mi}
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: archive
  hostPath: {path: /tmp/archive}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: archive-claim, namespace: l4-lab}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: archive
  resources: {requests: {storage: 100Mi}}
EOF
```

### Verification

```bash
kubectl -n l4-lab get pv,pvc
```

`app-data` is `Bound` to a dynamically-provisioned PV from the default SC (on the live cluster this is a real **Cinder** volume via `cinder.csi.openstack.org`; on kind it's `standard`/local-path); `archive-claim` is `Bound` to `archive-pv`; the value `reader` prints matches the writer's `$MARKER` — data survived the Pod restart.

> **Teaching note:** Both PV and PVC must share `storageClassName: archive` — omit it on the PVC and it falls through to the default SC and never binds the hand-rolled PV (stuck `Pending`). On the OpenStack cluster the default class binds with `WaitForFirstConsumer`, so the PV/Cinder volume is only created once the consuming Pod is scheduled — `get pvc` stays `Pending` until then, which is expected, not a bug.

---

## Lab 10 — Application Access (Octavia/Ingress/Gateway)

**⏱ 25 min**   🟡

### Solution

```bash
# --- Part 1: Service types + Ingress ---
kubectl create ns l5-lab
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: hello, namespace: l5-lab}
spec:
  replicas: 2
  selector: {matchLabels: {app: hello}}
  template:
    metadata: {labels: {app: hello}}
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args: ["-listen=:5678", "-text=hello-from-pod"]
        ports: [{containerPort: 5678}]
EOF
kubectl -n l5-lab wait --for=condition=Available deployment/hello

kubectl -n l5-lab expose deployment hello --name=hello    --port=80 --target-port=5678
kubectl -n l5-lab expose deployment hello --name=hello-np --port=80 --target-port=5678 --type=NodePort
# On the OpenStack cluster you can also expose --type=LoadBalancer → Octavia floating IP.

kubectl -n l5-lab run loop --image=busybox:1.37 --rm -it --restart=Never -- \
  sh -c 'for i in 1 2 3 4 5; do wget -qO- http://hello/; done'

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: hello-ing, namespace: l5-lab}
spec:
  ingressClassName: nginx
  rules:
  - host: hello.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: hello, port: {number: 80}}}
EOF

# --- Part 2: Gateway API (CRDs not built in) ---
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
kubectl create ns gw-lab
kubectl -n gw-lab create deployment site --image=hashicorp/http-echo:1.0 \
  -- /http-echo -listen=:5678 -text="routed-by-gateway"
kubectl -n gw-lab expose deployment site --port=80 --target-port=5678
cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata: {name: lab-class}
spec: {controllerName: example.com/gateway-controller}
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: {name: lab-gw, namespace: gw-lab}
spec:
  gatewayClassName: lab-class
  listeners: [{name: http, protocol: HTTP, port: 80}]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: site-route, namespace: gw-lab}
spec:
  parentRefs: [{name: lab-gw}]
  hostnames: ["site.lab"]
  rules:
  - matches: [{path: {type: PathPrefix, value: /}}]
    backendRefs: [{name: site, port: 80}]
EOF
```

### Verification

```bash
kubectl -n l5-lab get svc,ingress,endpoints
INGRESS_IP=$(kubectl -n ingress-nginx get pod \
  -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].status.podIP}')
kubectl run t --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  curl -s -H 'Host: hello.local' http://$INGRESS_IP/      # -> hello-from-pod

kubectl get crd | grep -c gateway.networking.k8s.io        # 5
kubectl -n gw-lab get gateway,httproute
```

`hello` is a ClusterIP with 2 endpoints; `hello-np` a NodePort with an assigned port; the ingress curl returns `hello-from-pod`. The 5 Gateway-API CRDs exist; the `HTTPRoute` binds `lab-gw → site:80`, but the Gateway shows **no `ADDRESS`** and `PROGRAMMED=Unknown` because no controller implements `example.com/gateway-controller`.

> **Teaching note:** `--target-port=5678` is essential — http-echo listens on 5678; expose only `--port=80` and you get connection-refused even though the Service looks healthy. The Gateway lab is the Gateway-API version of "an Ingress does nothing without an ingress controller": the three kinds map to three roles (GatewayClass = infra, Gateway = cluster operator, HTTPRoute = app dev). On this OpenStack cluster, `type: LoadBalancer` is the real production entry point — Octavia hands the Service a floating IP, which is the cloud-native path the ingress-nginx controller itself uses.

---

## Lab 11 — Configuration & Quotas

**⏱ 15 min**   🟢

### Solution

```bash
kubectl create ns dev

# LimitRange — per-Container defaults
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata: {name: defaults, namespace: dev}
spec:
  limits:
  - type: Container
    default:        {cpu: "500m", memory: "512Mi"}
    defaultRequest: {cpu: "100m", memory: "128Mi"}
EOF

# ResourceQuota — namespace caps
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata: {name: cap, namespace: dev}
spec:
  hard:
    pods: "4"
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
EOF

# ConfigMap (literal + multiline key) + Secret
kubectl -n dev create configmap app-config \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=app.properties=$'feature.x=true\nfeature.y=false'
kubectl -n dev create secret generic app-secret --from-literal=api_key=ABCD-1234

# Deployment: env from CM, single CM key mounted as a file (subPath+items), Secret dir
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: webapp, namespace: dev}
spec:
  replicas: 1
  selector: {matchLabels: {app: webapp}}
  template:
    metadata: {labels: {app: webapp}}
    spec:
      containers:
      - name: web
        image: nginx:1.27
        env:
        - name: LOG_LEVEL
          valueFrom: {configMapKeyRef: {name: app-config, key: LOG_LEVEL}}
        volumeMounts:
        - {name: cfg, mountPath: /etc/app/app.properties, subPath: app.properties}
        - {name: sec, mountPath: /etc/app/secret, readOnly: true}
      volumes:
      - name: cfg
        configMap:
          name: app-config
          items: [{key: app.properties, path: app.properties}]
      - name: sec
        secret: {secretName: app-secret}
EOF
```

### Verification

```bash
kubectl -n dev describe resourcequota cap
POD=$(kubectl -n dev get pod -l app=webapp -o name | head -1)
kubectl -n dev exec $POD -- env | grep LOG_LEVEL          # LOG_LEVEL=debug
kubectl -n dev exec $POD -- cat /etc/app/app.properties   # feature.x=true / feature.y=false
kubectl -n dev exec $POD -- cat /etc/app/secret/api_key   # ABCD-1234
```

Container resources are auto-injected by the LimitRange: requests `100m/128Mi`, limits `500m/512Mi`.

> **Teaching note:** Mounting a single key as a file needs `subPath` + `items` — without `subPath` the mount replaces the whole `/etc/app/` directory with the ConfigMap. Once a ResourceQuota with `requests.cpu` is active, a Pod with no requests is rejected — which is exactly why the LimitRange (auto-injecting defaults) is paired with it. Secrets are base64, **not** encrypted at rest by default — see Lab 18/21 for `kube_encrypt_secret_data`.

---

## Lab 12 — Scheduling

**⏱ 20 min**   🟡

### Solution

```bash
kubectl create ns l8-lab
kubectl label node cka-lab-worker  role=web   --overwrite
kubectl label node cka-lab-worker2 role=batch --overwrite
kubectl taint nodes cka-lab-worker2 workload=batch:NoSchedule --overwrite

# batch-job → worker2 (nodeSelector role=batch + toleration)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: batch-job, namespace: l8-lab}
spec:
  replicas: 1
  selector: {matchLabels: {app: batch-job}}
  template:
    metadata: {labels: {app: batch-job}}
    spec:
      nodeSelector: {role: batch}
      tolerations: [{key: workload, operator: Equal, value: batch, effect: NoSchedule}]
      containers: [{name: job, image: busybox:1.37, command: ["sh","-c","sleep 600"]}]
EOF

# web v1: role=web + anti-affinity → only one node has role=web → one Pod Pending
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: web, namespace: l8-lab}
spec:
  replicas: 2
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      nodeSelector: {role: web}
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector: {matchLabels: {app: web}}
            topologyKey: kubernetes.io/hostname
      containers: [{name: nginx, image: nginx:1.27}]
EOF

# web v2: broaden selector to (web,batch) + tolerate the batch taint.
# Delete+recreate avoids the surge deadlock a rolling update would hit.
kubectl -n l8-lab delete deployment web
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: web, namespace: l8-lab}
spec:
  replicas: 2
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      tolerations: [{key: workload, operator: Equal, value: batch, effect: NoSchedule}]
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions: [{key: role, operator: In, values: [web, batch]}]
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector: {matchLabels: {app: web}}
            topologyKey: kubernetes.io/hostname
      containers: [{name: nginx, image: nginx:1.27}]
EOF
kubectl -n l8-lab rollout status deployment/web

# stowaway: targets batch node WITHOUT toleration → stays Pending
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: stowaway, namespace: l8-lab}
spec:
  nodeSelector: {role: batch}
  containers: [{name: app, image: nginx:1.27}]
EOF
sleep 5
kubectl -n l8-lab describe pod stowaway | grep -A1 'Events:'
```

### Verification

```bash
kubectl -n l8-lab get pod -o wide
# batch-job-* → cka-lab-worker2 ; web-* (2) → one per worker ;
# stowaway → Pending, "1 node(s) had untolerated taint {workload: batch}"
```

> **Teaching note:** In the v2 step a rolling update would surge a 3rd Pod that can't satisfy the 1-per-node anti-affinity and deadlock — delete+recreate (or `maxSurge: 0`) is the clean fix; students who `kubectl edit` often hang on `rollout status`. Stress that `nodeSelector` targets a node but does **not** tolerate a taint — both are needed to land on a tainted, labeled node. Clean up the node labels/taints afterward or later labs misbehave.

---

## Lab 13 — Networking & NetworkPolicy

**⏱ 20 min**   🟡

### Solution

```bash
kubectl create ns l9-lab

# Per tier: echo server + Service + busybox tester, all sharing the `tier` label
for t in web api db; do
  kubectl -n l9-lab run ${t}-svc --image=hashicorp/http-echo:1.0 \
    --labels=tier=$t,role=server --port=5678 -- -listen=:5678 -text="hello from $t"
  kubectl -n l9-lab expose pod ${t}-svc --name=$t --port=80 --target-port=5678
  kubectl -n l9-lab run ${t}-tester --image=busybox:1.37 \
    --labels=tier=$t,role=tester --command -- sh -c "sleep infinity"
done
kubectl -n l9-lab wait --for=condition=Ready pod -l tier --timeout=60s

# baseline (all allowed) — then lock down
kubectl -n l9-lab exec web-tester -- wget -qO- -T 3 http://api/

# default-deny ingress
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny, namespace: l9-lab}
spec: {podSelector: {}, policyTypes: [Ingress]}
EOF

# api accepts only from tier=web ; db accepts only from tier=api (target port 5678!)
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: api-from-web, namespace: l9-lab}
spec:
  podSelector: {matchLabels: {tier: api}}
  policyTypes: [Ingress]
  ingress:
  - from: [{podSelector: {matchLabels: {tier: web}}}]
    ports: [{port: 5678, protocol: TCP}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: db-from-api, namespace: l9-lab}
spec:
  podSelector: {matchLabels: {tier: db}}
  policyTypes: [Ingress]
  ingress:
  - from: [{podSelector: {matchLabels: {tier: api}}}]
    ports: [{port: 5678, protocol: TCP}]
EOF
```

### Verification

```bash
kubectl -n l9-lab exec web-tester -- wget -qO- -T 5 http://api/   # "hello from api"
kubectl -n l9-lab exec web-tester -- wget -qO- -T 5 http://db/    # download timed out
kubectl -n l9-lab exec api-tester -- wget -qO- -T 5 http://db/    # "hello from db"
```

> **Teaching note:** The policy `port` is the **target/container** port (5678), not the Service port (80) — write `port: 80` and everything is blocked even from allowed sources. Policies are **additive and deny-by-default once any policy selects a Pod**. This requires a policy-enforcing CNI: on the OpenStack cluster that's **Calico** (enforces by default); kindnet does **not**, so if you demo locally on plain kind the deny won't take effect.

---

## Lab 14 — Security (RBAC)

**⏱ 20 min**   🟡

### Solution

```bash
kubectl create ns team-a
kubectl -n team-a create serviceaccount read-only-operator

# built-in `view` ClusterRole, namespaced via RoleBinding
kubectl -n team-a create rolebinding view-in-team-a \
  --clusterrole=view --serviceaccount=team-a:read-only-operator

# nodes are cluster-scoped → need a ClusterRole + ClusterRoleBinding
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: {name: nodes-viewer}
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get","list","watch"]
EOF
kubectl create clusterrolebinding nodes-viewer-binding \
  --clusterrole=nodes-viewer --serviceaccount=team-a:read-only-operator

# prove the limits without a Pod
SA=system:serviceaccount:team-a:read-only-operator
kubectl -n team-a  auth can-i list pods          --as=$SA   # yes
kubectl -n team-a  auth can-i create deployments --as=$SA   # no
kubectl            auth can-i list nodes         --as=$SA   # yes
kubectl -n default auth can-i list pods          --as=$SA   # no

# run a Pod under the SA (no --serviceaccount flag on run → manifest)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: kctl, namespace: team-a}
spec:
  serviceAccountName: read-only-operator
  containers: [{name: kc, image: bitnami/kubectl:latest, command: ["sh","-c","sleep infinity"]}]
EOF
kubectl -n team-a wait --for=condition=Ready pod/kctl --timeout=60s
kubectl -n team-a exec kctl -- kubectl get pods -n team-a    # OK
kubectl -n team-a exec kctl -- kubectl get pods -n default   # Forbidden
kubectl -n team-a exec kctl -- kubectl get nodes             # OK
```

### Verification

`auth can-i` answers: `yes / no / yes / no`. From the Pod, `get pods -n team-a` and `get nodes` succeed (the latter lists the 3 cluster nodes `Ready`); `get pods -n default` returns a Forbidden error for `system:serviceaccount:team-a:read-only-operator`.

> **Teaching note:** `nodes` are cluster-scoped — a RoleBinding can't grant access to them, so a Role/RoleBinding for nodes silently fails `can-i`; a ClusterRole + ClusterRoleBinding is required. The reverse trap: binding a ClusterRole via a *RoleBinding* scopes it to one namespace (used here for `view`). `kubectl run` has no `--serviceaccount` flag — use a manifest with `serviceAccountName`.

---

## Lab 15 — CRDs & Operators

**⏱ 15 min**   🟢

### Solution

```bash
# define the CRD (metadata.name MUST be <plural>.<group>)
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: {name: widgets.demo.example.com}
spec:
  group: demo.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:  {type: string}
              count: {type: integer}
    additionalPrinterColumns:
    - {name: Size,  type: string,  jsonPath: .spec.size}
    - {name: Count, type: integer, jsonPath: .spec.count}
  scope: Namespaced
  names: {plural: widgets, singular: widget, kind: Widget, shortNames: ["wd"]}
EOF

# create a custom resource
cat <<'EOF' | kubectl apply -f -
apiVersion: demo.example.com/v1
kind: Widget
metadata: {name: blue-widget}
spec: {size: large, count: 5}
EOF

kubectl get wd                       # printer columns: SIZE / COUNT
kubectl explain widget.spec
kubectl api-resources | grep demo.example.com
```

**Operator answer:** add a **custom controller** — a Deployment running a reconcile loop that watches `Widget` objects and drives the cluster toward their `.spec`. CRD + controller = an **operator**.

### Verification

`kubectl get crd widgets.demo.example.com` is Established; `kubectl get wd blue-widget` prints `blue-widget   large   5` via the `additionalPrinterColumns`; `kubectl explain widget.spec` lists `size`, `count`. The CRD object is **cluster-scoped** even though the `Widget` it defines is namespaced.

> **Teaching note:** Three exam favourites: the CRD `metadata.name` **must** equal `<plural>.<group>`; exactly **one** version is `storage: true`; and a structural `openAPIV3Schema` is **mandatory** in `apiextensions.k8s.io/v1`. After installing an operator (cert-manager, Prometheus Operator, …) you discover its new kinds with `kubectl get crd` / `kubectl api-resources` and read their schema with `kubectl explain` — then drive it with custom resources, not low-level objects.

---

## Lab 16 — Node Maintenance & Scaling

**⏱ 35 min**   🔴   🖥️ *(Ansible control node + fleet)*

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray
INV=inventory/mycluster/inventory.ini

# --- Part A: add a worker (node6) ---
# 1 (🖥️) current state
kubectl get nodes
# 2 append node6 to [all] + [kube_node] ONLY (not etcd / kube_control_plane)
ansible-inventory -i "$INV" --graph kube_node          # confirm node6 landed in kube_node
# 3 (🖥️) refresh facts for ALL nodes first (no --limit) so the limited run isn't stale
ansible-playbook -i "$INV" -b playbooks/facts.yml
# 4 (✅) syntax check
ansible-playbook -i "$INV" --syntax-check playbooks/scale.yml
# 5 (🖥️) join ONLY node6 (additive, ~5–10 min)
ansible-playbook -i "$INV" -u ubuntu -b --private-key ~/.ssh/id_rsa \
  --limit=node6 playbooks/scale.yml
# 6 verify
kubectl get nodes

# --- Part B: remove a node (node5) ---
# 7 (🖥️) remove WHILE still in inventory (cordons + drains + resets it)
ansible-playbook -i "$INV" -b -e node=node5 playbooks/remove_node.yml
#   offline node? add: -e reset_nodes=false -e allow_ungraceful_removal=true
# 8 NOW delete node5 from [all] and [kube_node] in the inventory
# 9 verify
kubectl get nodes
ansible-inventory -i "$INV" --graph kube_node          # node5 gone

# --- Part C: special cases (📖) ---
# Control-plane node → cluster.yml (NOT scale.yml), append to END of [kube_control_plane],
#   then restart nginx-proxy on every host. etcd stays ODD. Never run HA below 3 CP nodes.
```

### Verification

**✅ Control node:** `--syntax-check` of `scale.yml` and `remove_node.yml` both exit 0; `ansible-inventory --graph kube_node` shows `node6` appended (and, after Part B, `node5` removed).

**🖥️ Against the live cluster:**

```text
# after Part A (scale.yml --limit=node6): node6 joins Ready
node6   Ready    <none>          1m    v1.31.x     # <-- added
# after Part B (remove_node.yml -e node=node5): node5 gone, node6 remains
```

The control plane stays at **3** throughout — only worker capacity changed.

> **Teaching note:** `scale.yml` is the **additive worker path** (joins new workers without re-running the whole cluster); never use it for a control-plane node — use `cluster.yml`. **Order matters** on removal: run `remove_node.yml` *first* (it needs the node still in the inventory to cordon/drain/reset), *then* edit it out. Always run `facts.yml` (no `--limit`) before any `--limit` scale, or the limited run works from stale facts. Never drop an HA cluster below **3** control-plane/etcd members.

---

## Lab 17 — Upgrades (`upgrade-cluster.yml`)

**⏱ 40 min**   🔴   🖥️ *(Ansible control node + fleet)*

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray

# 1 (🖥️) note the starting version
kubectl get nodes -o wide ; kubectl version

# 2 (🖥️) etcd snapshot FIRST — your ONLY rollback path (run on a control-plane node)
ssh ubuntu@<cp1-ip> 'sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-$(hostname).pem --key=/etc/ssl/etcd/ssl/admin-$(hostname)-key.pem \
  snapshot save /var/backups/etcd-pre-upgrade-$(date +%F).db'

# 3 move Kubespray ONE tag forward (one tag at a time)
git fetch --tags && git checkout v2.27.0

# 4 reinstall pinned deps, then diff group_vars for drift
pip install -r requirements.txt
diff -ruq inventory/sample/group_vars inventory/mycluster/group_vars | less

# 5 bump kube_version to the NEXT minor (one minor at a time): v1.30.x -> v1.31.4

# 6 (✅) syntax check
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/upgrade_cluster.yml

# 7 (🖥️) graceful upgrade, ONE node at a time (cordon/drain/uncordon per node)
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa -e kube_version=v1.31.4 -e serial=1 \
  playbooks/upgrade_cluster.yml

# 8 verify, then REPEAT 3–7 for 1.31 -> 1.32
kubectl get nodes ; kubectl version
```

Handy knobs: `-e serial=1` (one node at a time), `--limit "node4:node6"`, `-e upgrade_node_confirm=true` (manual approval per node), `-e upgrade_node_pause_seconds=60`.

### Verification

**✅ Control node:** `--syntax-check playbooks/upgrade_cluster.yml` exits 0.

**🖥️ During/after the run** — with `serial=1`, nodes flip one at a time and the cluster stays serviceable:

```text
# mid-roll: node1 done, node2 cordoned, node3 not yet
node1   Ready                      control-plane   v1.31.4
node2   Ready,SchedulingDisabled   control-plane   v1.30.4
node3   Ready                      <none>          v1.30.4
# after this minor: all on v1.31.4 ; Server Version: v1.31.4
```

Confirm `get nodes` all-`Ready` and `get pods -A` healthy **between each minor**.

> **Teaching note:** `upgrade_cluster.yml` ≠ `cluster.yml` — the upgrade playbook **cordons, drains, uncordons** each node; `cluster.yml` does **not** drain. Two hard rules: **never skip a minor** (`1.30 → 1.31 → 1.32`, never `1.30 → 1.32` — kubeadm version skew) and **move Kubespray one tag at a time** (tag and K8s range are coupled). There is **no clean in-place rollback** — the etcd snapshot from step 2 is the only recovery path. group_vars formats drift between tags, so diff the sample for the new tag before running.

---

## Lab 18 — Backup, Recovery & Reset

**⏱ 35 min**   🔴   🖥️ *(Ansible control node + fleet)*

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray

# (✅) syntax-check both playbooks before touching anything
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/recover_control_plane.yml
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/reset.yml

# 1–2 (🖥️) snapshot on an etcd node (host deployment → certs under /etc/ssl/etcd/ssl/)
sudo ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd_snapshot.db \
  --endpoints=https://127.0.0.1:2379 --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-<host>.pem --key=/etc/ssl/etcd/ssl/admin-<host>-key.pem

# 3 (🖥️) verify the snapshot, then copy it OFF the node
sudo ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd_snapshot.db --write-out=table
scp <user>@<etcd-node-ip>:/tmp/etcd_snapshot.db /tmp/etcd_snapshot.db

# 5 (📖) recovery inventory: broken_etcd / broken_kube_control_plane groups, survivors FIRST
# 6 (🖥️) recover — scope to broken + a healthy node; pass YOUR snapshot for lost-quorum
ansible-playbook -i inventory/mycluster/hosts.yaml -b \
  --limit etcd,kube_control_plane \
  -e etcd_retries=10 -e etcd_snapshot=/tmp/etcd_snapshot.db \
  playbooks/recover_control_plane.yml

# 7 (🖥️) tear down (destructive — back up etcd first!)
ansible-playbook -i inventory/mycluster/hosts.yaml -b -e reset_confirmation=yes playbooks/reset.yml

# 8 (📖) encryption at rest toggle in k8s-cluster.yml:
#   kube_encrypt_secret_data: true   # default false; provider = secretbox
```

### Verification

**✅ Control node:** both `--syntax-check` runs exit 0.

**🖥️ After `snapshot save`** the success line prints and `snapshot status --write-out=table` shows integrity hash, revision, key count and size:

```text
Snapshot saved at /tmp/etcd_snapshot.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 7d2c1f0a |   154832 |       1394 |     4.7 MB |
+----------+----------+------------+------------+
```

(Values vary; what matters is a non-zero `TOTAL KEYS`/`TOTAL SIZE` — an empty table means a corrupt snapshot.) After recovery, the broken etcd member rejoins (`etcdctl member list` / `endpoint health`) and `kubectl get nodes` / `get pods -A` are healthy. After `reset.yml`, the nodes have no kubelet/etcd/CNI left and are ready for a fresh `cluster.yml`.

> **Teaching note:** Never trust a backup you haven't inspected — `snapshot status` is the gate. The `host` etcd deployment keeps certs under `/etc/ssl/etcd/ssl/` (note the `admin-<host>` filenames); a kubeadm/static-Pod etcd uses `/etc/kubernetes/pki/etcd/` instead. `recover_control_plane.yml` **detects quorum**: with quorum it rebuilds members; with quorum lost it restores from `-e etcd_snapshot=`. The docs warn it's only tested on small DBs and gives no guarantees — rehearse on a throwaway cluster (stop etcd + `rm -rf /var/lib/etcd/member`, then recover). Secrets are base64, not encrypted, so a snapshot exposes them unless `kube_encrypt_secret_data` is on.

---

## Lab 19 — Add-ons & Packaging

**⏱ 25 min**   🟡

### Solution

```bash
# --- Part 1: Kubespray add-ons (group_vars/k8s_cluster/addons.yml) ---
# Simple toggles:
#   metrics_server_enabled / helm_enabled / cert_manager_enabled
#   local_volume_provisioner_enabled / registry_enabled
# MetalLB + address pool:
cat <<'YAML'
metallb_enabled: true
metallb_speaker_enabled: true
metallb_namespace: "metallb-system"
metallb_config:
  address_pools:
    primary:
      ip_range: ["192.168.121.240-192.168.121.250"]
      auto_assign: true
      avoid_buggy_ips: true
  layer2: [primary]
# k8s-cluster.yml prerequisite for MetalLB L2:
#   kube_proxy_strict_arp: true
YAML

# (✅) validate YAML, then apply add-ons (full cluster.yml or just --tags=apps)
python3 -c "import yaml; yaml.safe_load(open('inventory/mycluster/group_vars/k8s_cluster/addons.yml')); print('addons.yml: valid YAML')"
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa --tags=apps playbooks/cluster.yml      # 🖥️

# Verify each add-on (🖥️): metrics, MetalLB pool, and a real LoadBalancer Service
kubectl top nodes
kubectl get ipaddresspool -A
kubectl create deployment nginx --image=nginx --port=80
kubectl expose deployment nginx --type=LoadBalancer --port=80 --name=nginx-lb
kubectl get svc nginx-lb -w     # EXTERNAL-IP flips from <pending> to a pool IP

# --- Part 2: Helm + Kustomize (usage) ---
helm create site
helm install site ./site -n helm-lab --create-namespace --set replicaCount=3
helm upgrade site ./site -n helm-lab --set replicaCount=3 --set image.tag=1.27
helm rollback site 1 -n helm-lab            # back to revision 1 (replicaCount=3)
helm list -n helm-lab ; helm history site -n helm-lab

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

**Add-ons (✅ + 🖥️):** `addons.yml: valid YAML`; the address pool round-trips to `['192.168.121.240-192.168.121.250']`, `layer2 = ['primary']`. After `--tags=apps`: `metrics-server`, `cert-manager*`, `registry*`, `local-volume-provisioner*`, and `metallb-system` (`controller` + `speaker`) Pods `Running`; `kubectl top nodes` returns CPU/MEM; `kubectl get ipaddresspool -A` lists `primary`; `kubectl get svc nginx-lb` shows an `EXTERNAL-IP` from the pool (no longer `<pending>`).

**Packaging:** `helm list -n helm-lab` shows release `site` **deployed**; `kubectl -n helm-lab get deploy site -o jsonpath='{.spec.replicas}'` → **3** after rollback to revision 1. `kubectl -n kz-staging get deploy -o wide` shows **`stg-web`** at **2/2** with image **`nginx:1.27-alpine`** — name prefix, namespace, replica patch and image override, no templating.

> **Teaching note:** Recent Kubespray releases **dropped some in-tree add-ons** (ingress-nginx, Dashboard) — install those **with Helm post-deploy** (which is why `helm_enabled` is on); keep cert-manager / registry / MetalLB / local-PVs as in-tree toggles. On the OpenStack cluster, `type: LoadBalancer` is already served by **Octavia**, so MetalLB is mainly for non-cloud demos — don't run both for the same Service. Helm value precedence: `--set` beats `-f` files beat the chart's `values.yaml`; `local_volume_provisioner` is *static* (`no-provisioner`) and only exposes mounts that already exist.

---

## Lab 20 — Troubleshooting

**⏱ 20 min**   🟡

### Solution

Setup deploys three broken Deployments in `l11-lab` (`broken-image`, `crashloop`, `unschedulable`). Diagnose from `kubectl` alone, then fix with the smallest change:

```bash
kubectl -n l11-lab get pod

# root causes
kubectl -n l11-lab describe pod -l app=broken-image | sed -n '/^Events:/,$p'
# → "Failed to pull image nginx:9.99-nope ... not found"
kubectl -n l11-lab logs -l app=crashloop --tail=20
# → prints "starting" then exits → CrashLoopBackOff
kubectl -n l11-lab describe pod -l app=unsched | grep -A3 'Events:'
# → "0/3 nodes are available: ... didn't match Pod's node affinity/selector"

# minimal fixes
kubectl -n l11-lab set image deployment/broken-image nginx=nginx:1.27
kubectl -n l11-lab patch deployment crashloop --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/command","value":["sh","-c","echo healthy; sleep infinity"]}
]'
kubectl -n l11-lab patch deployment unschedulable --type=json -p='[
  {"op":"remove","path":"/spec/template/spec/nodeSelector"}
]'
kubectl -n l11-lab rollout status deployment/broken-image
kubectl -n l11-lab rollout status deployment/crashloop
kubectl -n l11-lab rollout status deployment/unschedulable
```

### Verification

```bash
kubectl -n l11-lab get pod   # all 1/1 Running, stable restart count
```

> **Teaching note:** Teach the triage order: `get pod` (status) → `describe pod` (Events at the bottom, for scheduling/image pulls) → `logs` / `logs --previous` (for crashes). The container name from `create deployment broken-image --image=nginx` is the *deployment name*, not `nginx` — but `set image deployment/broken-image nginx=...` still works because that create form names the container after the image's first segment; have students confirm with `-o jsonpath='{.spec.template.spec.containers[0].name}'` before `set image`. On the real cluster, also reach for host-level checks (`journalctl -u kubelet`, `kubectl describe node`) and `kubectl debug` for distroless images.

---

## Lab 21 — Hardening & Offline

**⏱ 20 min**   🟢

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray

# 1 (✅) apply the CI hardening scenario as your k8s_cluster group_vars base
cp tests/files/ubuntu24-calico-all-in-one-hardening.yml \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
#   It turns on: authorization_modes [Node,RBAC], remove_anonymous_access,
#   kubernetes_audit + audit_log_path, kube_encrypt_secret_data (secretbox),
#   kube_profiling: false, PodSecurity + EventRateLimit admission plugins,
#   PSA restricted enforce, kube_read_only_port: 0, kubelet rotate/seccomp/systemd hardening,
#   controller-manager & scheduler bind 127.0.0.1.

# 2 (✅) YAML valid + key values as expected
python3 -c "import yaml; d=yaml.safe_load(open('inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml')); \
assert d['remove_anonymous_access'] and d['kubernetes_audit'] and d['kube_encrypt_secret_data']; \
assert d['kube_read_only_port']==0 and d['kube_profiling'] is False; \
assert d['kube_pod_security_default_enforce']=='restricted'; \
assert 'PodSecurity' in d['kube_apiserver_enable_admission_plugins']; print('hardening keys OK')"

# 3 (✅) syntax-check with hardened vars in place
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml

# 4 (📖) large-deployment tuning: --forks/--timeout on the CLI; download_run_once,
#        etcd_events_cluster_setup, longer node-monitor/eviction timeouts, dns_replicas.
# 5 (📖) offline/air-gapped: group_vars/all/offline.yml (registry_host, files_repo,
#        *_image_repo, *_url, containerd mirror) + contrib/offline/generate_list.sh.

# 6 (🖥️) deploy hardened (--forks is the scale knob)
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa --forks 10 --timeout 600 playbooks/cluster.yml
```

### Verification

**✅ Control node:** `hardening keys OK`; the grep shows `remove_anonymous_access: true`, `kubernetes_audit: true`, `kube_encrypt_secret_data: true`, `kube_read_only_port: 0`, `kube_profiling: false`, `kube_pod_security_default_enforce: restricted`; `--syntax-check` exits 0.

**🖥️ After the hardened deploy** — prove each control:

```text
(a) /etc/kubernetes/manifests/kube-apiserver.yaml →
    --anonymous-auth=false  --profiling=false
    --enable-admission-plugins=...,PodSecurity,EventRateLimit,...
    --audit-log-path=/var/log/kube-apiserver-log.json
    --encryption-provider-config=/etc/kubernetes/ssl/secrets_encryption.yaml
(b) kubectl run nginx --image=nginx --privileged
    → Error from server (Forbidden): violates PodSecurity "restricted:latest": privileged
(c) /var/log/kube-apiserver-log.json exists and grows; tail → {"kind":"Event",...}
(e) ss -tlnp | grep ':10255'  → read-only port closed (good)
(f) ss -tlnp | grep -E ':10257|:10259'  → both 127.0.0.1 (loopback only)
```

Optionally run **kube-bench** for the full CIS report.

> **Teaching note:** These values map directly to **CIS Kubernetes Benchmark** checks — the scenario is the fast path to a kube-bench-clean cluster. The functional proof students should remember is (b): a privileged Pod is **Forbidden** by the `restricted` PSA default (kube-system is exempt by design). The CI file also sets `etcd_deployment_type: kubeadm` for conformance; the hardening docs prefer `host` to isolate etcd — pick per topology. `download_run_once: true` pairs naturally with the offline mirrors; keep registry/file-server passwords in **Ansible Vault**.

---

## Lab 22 — Cloud & OpenStack

**⏱ 25 min**   🟡   🖥️ *(Ansible control node + cloud)*

### Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray
cp -rfp inventory/sample inventory/mycluster   # ✅ gives group_vars/all/openstack.yml

# (📖) collect IDs with the OpenStack CLI:
source ~/openstack-rc
openstack network list ; openstack subnet list
openstack catalog list | grep -i load-balancer
openstack application credential create kubespray-ccm   # id + secret (shown once)

# 3 (✅) external cloud provider — group_vars/all/all.yml
#   cloud_provider: external
#   external_cloud_provider: openstack

# 4 (✅) auth + Octavia LBaaS — group_vars/all/openstack.yml
#   external_openstack_auth_url / region / application_credential_{name,id,secret}
#   external_openstack_lbaas_enabled: true ; provider: amphora
#   external_openstack_lbaas_floating_network_id / _floating_subnet_id / _subnet_id / _member_subnet_id

# 5 (✅) Cinder CSI + default StorageClass — same file
#   cinder_csi_enabled: true
#   storage_classes: [{name: cinder-csi, is_default: true,
#     provisioner: cinder.csi.openstack.org, volume_binding_mode: WaitForFirstConsumer, ...}]
#   k8s-cluster.yml: persistent_volumes_enabled: true

# 6 (✅) syntax-check
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml

# 7 (🖥️) source the rc, (re-)deploy the CCM + Cinder CSI
source ~/openstack-rc
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa playbooks/cluster.yml

# 8–10 (🖥️) verify CCM/CSI Pods, a LoadBalancer floating IP, a Cinder PVC
kubectl -n kube-system get pods | grep -E 'cloud-controller|cinder'
kubectl create deployment web --image=nginx --port=80
kubectl expose deployment web --type=LoadBalancer --port=80
kubectl get svc web -w        # EXTERNAL-IP → floating IP
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: csi-pvc-cinder}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: cinder-csi
  resources: {requests: {storage: 1Gi}}
EOF
```

### Verification

**✅ Control node:** `--syntax-check playbooks/cluster.yml` exits 0 (`openstack.yml` + `all.yml` parse).

**🖥️ Against the live cluster:**

```text
$ kubectl -n kube-system get pods | grep -E 'cloud-controller|cinder'
openstack-cloud-controller-manager-xxxxx   1/1   Running
csi-cinder-controllerplugin-xxxxxxxxx      6/6   Running
csi-cinder-nodeplugin-xxxxx                3/3   Running     # one per node

$ kubectl get storageclass
cinder-csi (default)   cinder.csi.openstack.org   Delete   WaitForFirstConsumer   true

$ kubectl get svc web
web   LoadBalancer   10.233.x.y   <floating-IP>   80:3xxxx/TCP

$ kubectl get pvc csi-pvc-cinder
csi-pvc-cinder   Bound   pvc-xxxx   1Gi   RWO   cinder-csi
```

Cross-check OpenStack-side: `openstack loadbalancer list` shows an amphora LB; `openstack volume list` shows a 1 GiB volume named after the PVC.

> **Teaching note:** In-tree cloud providers are **gone since K8s v1.31 / Kubespray v2.27** — `external` is the only supported value, and these two lines in `all.yml` are what cause `cluster.yml` to deploy the OpenStack CCM. Two traps: **inventory hostnames must equal the OpenStack instance names** or Cinder won't attach volumes; and for L3 CNIs (Calico) add **allowed-address-pairs** on the Neutron ports for the pod/service CIDRs or Services break. `external_openstack_lbaas_floating_network_id` is the var that gives a `LoadBalancer` Service its public floating IP. This is the capstone that ties inventory + group_vars + Day-2 ops together on the live cloud.

---

## Lab 23 — Practice Exam

**⏱ 60 min**   🔴

### Solution — Practice Exam #1

```bash
# 1.1 — Deployment + Service + NetworkPolicy (only role=tester may reach web)
kubectl create ns pe1-app
kubectl -n pe1-app create deployment web --image=nginx:1.27 --replicas=3
kubectl -n pe1-app label deployment web tier=frontend
kubectl -n pe1-app patch deployment web --type=merge -p \
  '{"spec":{"template":{"metadata":{"labels":{"tier":"frontend"}}}}}'
kubectl -n pe1-app expose deployment web --name=web --port=80
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: only-testers, namespace: pe1-app}
spec:
  podSelector: {matchLabels: {tier: frontend}}
  policyTypes: [Ingress]
  ingress:
  - from: [{podSelector: {matchLabels: {role: tester}}}]
    ports: [{port: 80, protocol: TCP}]
EOF

# 1.2 — Storage persistence (writer writes "done", reader on the same PVC reads it)
kubectl create ns pe1-storage
# PVC data(200Mi) + writer Pod → wait Ready → delete → reader Pod → logs reader == "done"
# (manifests as in Lab 9; reader command: cat /data/marker)

# 1.3 — Scheduling: payments on tier=gold, tolerating dedicated=batch, with requests/limits
kubectl label node cka-lab-worker  tier=gold --overwrite
kubectl taint nodes cka-lab-worker2 dedicated=batch:NoSchedule --overwrite
kubectl create ns pe1-sched
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: payments, namespace: pe1-sched}
spec:
  nodeSelector: {tier: gold}
  tolerations: [{key: dedicated, operator: Equal, value: batch, effect: NoSchedule}]
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests: {cpu: "100m", memory: "128Mi"}
      limits:   {cpu: "500m", memory: "512Mi"}
EOF

# 1.4 — Troubleshooting: remove the bad nodeSelector AND shrink the huge resources (BOTH)
kubectl -n pe1-broken describe pod -l app=broken | grep -A2 'Events:'
kubectl -n pe1-broken patch deployment broken --type=json -p='[
  {"op":"remove","path":"/spec/template/spec/nodeSelector"}
]'
kubectl -n pe1-broken patch deployment broken --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/resources","value":{"requests":{"cpu":"100m","memory":"128Mi"},"limits":{"cpu":"500m","memory":"512Mi"}}}
]'
kubectl -n pe1-broken rollout status deployment/broken
```

### Solution — Practice Exam #2

```bash
# 2.1 — RBAC: read-only in pe2-rbac + pe2-app ONLY (ClusterRole bound via RoleBindings)
kubectl create ns pe2-rbac ; kubectl create ns pe2-app
kubectl -n pe2-rbac create serviceaccount auditor
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: {name: read-only-auditor}
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get","list","watch"]
EOF
for ns in pe2-rbac pe2-app; do
  kubectl -n $ns create rolebinding auditor-in-$ns \
    --clusterrole=read-only-auditor --serviceaccount=pe2-rbac:auditor
done
# auditor-pod (bitnami/kubectl) under the SA → lists pods in pe2-rbac/pe2-app, Forbidden in default

# 2.2 — etcd snapshot (kubeadm/static-Pod etcd → certs under /etc/kubernetes/pki/etcd/)
ETCD_POD=$(kubectl -n kube-system get pod -l component=etcd -o name | head -1)
kubectl -n kube-system exec $ETCD_POD -- sh -c "
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/exam-snap.db && \
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /tmp/exam-snap.db"

# 2.3 — Node drain + uncordon
kubectl cordon cka-lab-worker2
kubectl drain  cka-lab-worker2 --ignore-daemonsets --delete-emptydir-data --force
kubectl get pod -A -o wide | awk '$8=="cka-lab-worker2"'   # only kube-system DS pods
kubectl uncordon cka-lab-worker2

# 2.4 — Ingress (Host: echo.local → echo:80, args -text=v2)
kubectl create ns pe2-ing
# Deployment echo (2 replicas) + Service echo(80→5678) + Ingress(class nginx) as in Lab 10
INGRESS_IP=$(kubectl -n ingress-nginx get pod \
  -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].status.podIP}')
kubectl run check --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  curl -s -H 'Host: echo.local' http://$INGRESS_IP/   # -> v2
```

### Verification

Exam #1: `pe1-app` has Deployment/Service/NetPol; `reader` logs `done`; `payments` schedules on the `tier=gold` node; `broken` becomes `1/1` Running. Exam #2: `auditor-pod` lists pods in `pe2-rbac`/`pe2-app` but is **Forbidden** in `default`; `snapshot status` prints a `HASH | REVISION | TOTAL KEYS | TOTAL SIZE` row; `cka-lab-worker2` returns `Ready` (no `SchedulingDisabled`) after uncordon; the ingress curl returns `v2`.

> **Teaching note:** The biggest time-sink is Task 1.4 — students fix only one of the two bugs (nodeSelector *or* resources) and the Deployment still won't roll out; teach them to read **all** of `describe` and apply both. Note the etcd cert path differs by deployment type: this exam's `2.2` assumes a kubeadm/static-Pod etcd (`/etc/kubernetes/pki/etcd/`), whereas Kubespray's default `host` etcd uses `/etc/ssl/etcd/ssl/` (Lab 18) — point students at the right paths for their cluster. Drill the exam habits: `alias k=kubectl`, `export do='--dry-run=client -o yaml'`, `kubectl explain`, `patch` over `edit`, and `auth can-i --as=` to verify RBAC without a Pod. Remind everyone to clean up node labels/taints between exams.
