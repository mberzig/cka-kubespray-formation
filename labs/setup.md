# Lab Setup — The Lab Cluster on OpenStack (via Kubespray)

> 🖥️ **All labs in this formation run on ONE shared cluster:** a **multi-node
> Kubernetes cluster provisioned on OpenStack VMs by Kubespray**. You need an
> OpenStack tenant (project) with quota for ~5–6 instances, **Octavia (LBaaS)**
> and **Cinder**. Procedures grounded in the Kubespray docs
> (`contrib/terraform/openstack`, `cloud_controllers/openstack`, `CSI/cinder-csi`).

## Why this environment

- **Infra labs** (build, HA, scaling, upgrade, etcd backup/recovery, hardening,
  cloud) need **real multi-node VMs** — they cannot run on `kind`.
- **Usage labs** (workloads, storage, networking, RBAC, …) run **identically** on
  this cluster — `kubectl` doesn't care that the nodes are OpenStack VMs.
- The **OpenStack integration** (external cloud provider + **Octavia** + **Cinder
  CSI**) makes `LoadBalancer` Services and **dynamic PVCs** work — used by the
  Access, Storage and Cloud labs.

## Topology

```
[ Ansible control node ]  ── SSH ──►  3 × control-plane (+ etcd)   ┐
  (your workstation /                  2 × worker                  ├─ OpenStack project
   a small bastion VM)                 floating IPs, private net,  │   (Octavia + Cinder)
   clone Kubespray, venv               security groups             ┘
```

A smaller variant (1 control-plane + 2 workers) is fine for everything except the
HA failover lab (Lesson 7), which needs 3 control planes.

## Prerequisites

- OpenStack tenant + **application credentials** (preferred) or `openrc`.
- Quota: ~6 instances, ~12 vCPU, ~24 GB RAM, ≥5 floating IPs, one Octavia LB,
  Cinder volume quota.
- An Ubuntu/Debian/Rocky image and a flavor (≥ 2 vCPU / 4 GB for control planes).
- An SSH keypair imported into OpenStack.

---

## Step 1 — Ansible control node

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
python3 -m venv venv && . venv/bin/activate
pip install -U pip && pip install -r requirements.txt
```

## Step 2 — Provision the VMs

### Path A (recommended) — Terraform creates the VMs *and* the inventory

Kubespray ships `contrib/terraform/openstack` to create instances / networks /
security groups and generate the dynamic inventory.

```bash
cp -LRp contrib/terraform/openstack/sample-inventory inventory/lab
cd inventory/lab
ln -s ../../contrib/terraform/openstack/hosts        # dynamic inventory
ln -s ../../contrib/terraform/openstack              # tf modules
# edit cluster.tfvars: cluster_name, image, ssh_user, external/internal network,
#   number_of_k8s_masters=3, number_of_k8s_nodes=2, flavors, floating IPs
source ~/openrc      # or export OS_* from application credentials
terraform -chdir=../../contrib/terraform/openstack init
terraform -chdir=../../contrib/terraform/openstack apply \
  -var-file=$PWD/cluster.tfvars
cd ../..
```

### Path B — manual VMs + hand-written inventory

```bash
openstack server create --image ubuntu-22.04 --flavor m1.large \
  --network lab-net --key-name lab --security-group k8s k8s-cp-{1,2,3}
openstack server create --image ubuntu-22.04 --flavor m1.large \
  --network lab-net --key-name lab --security-group k8s k8s-node-{1,2}
# assign floating IPs, then build inventory/lab/inventory.ini:
#   [kube_control_plane] cp1 cp2 cp3   [etcd] cp1 cp2 cp3   [kube_node] node1 node2
```

## Step 3 — Configure the OpenStack cloud integration (group_vars)

`inventory/lab/group_vars/all/all.yml`:

```yaml
cloud_provider: external
external_cloud_provider: openstack
cinder_csi_enabled: true
persistent_volumes_enabled: true
```

`inventory/lab/group_vars/all/openstack.yml` (Octavia + auth):

```yaml
external_openstack_auth_url: "https://keystone.example:5000/v3"
external_openstack_application_credential_id: "<id>"
external_openstack_application_credential_secret: "<secret>"
external_openstack_region: "RegionOne"
external_openstack_lbaas_subnet_id: "<private-subnet-id>"      # Octavia
external_openstack_lbaas_floating_network_id: "<ext-net-id>"
```

## Step 4 — Deploy the cluster

```bash
ansible-playbook -i inventory/lab/hosts -b cluster.yml      # ~20–40 min
# kubeconfig_localhost defaults to true -> inventory/lab/artifacts/admin.conf
export KUBECONFIG=$PWD/inventory/lab/artifacts/admin.conf
kubectl get nodes -o wide
```

## Step 5 — Verify (this is the baseline every lab assumes)

```bash
kubectl get nodes                       # 3 control-plane + 2 workers, all Ready

# Octavia LoadBalancer works:
kubectl create deploy web --image=nginx:1.27
kubectl expose deploy web --port=80 --type=LoadBalancer
kubectl get svc web -w                  # EXTERNAL-IP becomes a floating IP

# Cinder CSI dynamic provisioning works:
kubectl get sc                          # cinder-csi (cinder.csi.openstack.org)
kubectl delete deploy/web svc/web
```

**Expected:** all nodes `Ready`; the `LoadBalancer` Service receives a floating IP
from **Octavia**; the `cinder-csi` StorageClass provisions a **Cinder** volume for
a PVC.

---

## How the labs use this cluster

| Lab kind | Lessons | Run from | Notes |
|----------|---------|----------|-------|
| **Infra** | L2–L7, L16–L18, L21–L22 | the **Ansible control node** against `inventory/lab/` | inventory/group_vars edits, `cluster.yml`, `scale.yml`, `remove-node.yml`, `upgrade-cluster.yml`, `recover-control-plane.yml`, hardening, OpenStack integration |
| **Usage** | L8–L15, L19–L20 | **`kubectl`** against `admin.conf` | identical to any cluster; the OpenStack LB/CSI back the Access & Storage labs |

> Keep the control node's `KUBECONFIG` pointed at `inventory/lab/artifacts/admin.conf`
> and the venv activated — every lab starts from there.

## Cleanup

```bash
# tear the cluster down but keep the VMs:
ansible-playbook -i inventory/lab/hosts -b reset.yml -e reset_confirmation=yes
# destroy the OpenStack resources (Path A):
terraform -chdir=contrib/terraform/openstack destroy -var-file=$PWD/inventory/lab/cluster.tfvars
```
