# Lesson 22 — Cloud & OpenStack — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Take a working Kubespray inventory and wire the cluster into an **OpenStack**
tenant the modern, supported way: the **external cloud-controller-manager**
(`cloud_provider: external` + `external_cloud_provider: openstack`), Keystone
**application credentials**, **Octavia** for `LoadBalancer` Services, and the
**Cinder CSI** driver for dynamic block storage. Then (🖥️) re-run `cluster.yml`
and verify the three things that prove the integration works: the
cloud-controller and cinder-csi Pods are `Running`, a `type: LoadBalancer`
Service gets an Octavia **floating IP**, and a PVC provisions a real **Cinder
volume**. This is the capstone — it ties together inventory, group_vars and
Day-2 operations on a live cloud.

## Prerequisites

- Control node set up (`setup.md`): venv + `pip install -r requirements.txt`,
  and an inventory you can already deploy (Lessons 2 & 6).
- 📖 An **OpenStack tenant** with: Keystone v3 (`OS_AUTH_URL`), a project/region,
  Neutron with a **floating-IP (external) network**, **Octavia** LBaaS, and
  **Cinder v3** (the CSI driver supports v3 only).
- 📖 The VMs that will become nodes — provisioned either by hand or with
  `contrib/terraform/openstack`. **Inventory hostnames must be identical to the
  OpenStack instance names**, or Cinder will not attach volumes correctly.
- 📖 Reference docs:
  [OpenStack cloud-controller](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/cloud_controllers/openstack.md) ·
  [Cinder CSI](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/CSI/cinder-csi.md) ·
  [Cloud providers](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/cloud_providers/cloud.md) ·
  the annotated
  [`group_vars/all/openstack.yml`](https://github.com/kubernetes-sigs/kubespray/blob/master/inventory/sample/group_vars/all/openstack.yml)

## Steps (énoncé)

1. (📖, optional) Provision the VM fleet on OpenStack with
   `contrib/terraform/openstack`.
2. (✅) Start from a fresh inventory copy and gather the OpenStack IDs you'll need.
3. (✅) Enable the **external cloud provider** in `group_vars/all/all.yml`.
4. (✅) Fill **auth + Octavia LBaaS** in `group_vars/all/openstack.yml`
   (application credentials, auth URL/region, floating/subnet network IDs,
   amphora provider).
5. (✅) Enable **Cinder CSI** and a default `storage_classes` entry.
6. (✅) `--syntax-check` `cluster.yml` with the new vars in place.
7. (📖/🖥️) **Source your openstack-rc**, then (re-)run `cluster.yml` to roll out
   the CCM and Cinder CSI.
8. (🖥️) Verify the **cloud-controller** and **cinder-csi** Pods are `Running`.
9. (🖥️) Verify a `type: LoadBalancer` Service gets an **Octavia floating IP**.
10. (🖥️) Verify a **PVC** binds and provisions a **Cinder volume**.

## Solution

### 1 — (📖, optional) Provision VMs with Terraform

```bash
# 📖 needs the OpenStack tenant + terraform. Run from the repo:
cd contrib/terraform/openstack
cp cluster.tfvars.example cluster.tfvars     # edit: image, flavors, network, keypair, counts
source ~/openstack-rc                         # OS_* creds for the provider
terraform init
terraform apply -var-file=cluster.tfvars      # creates instances + writes inventory/hosts
# It generates a Kubespray-compatible inventory whose hostnames == instance names.
```

> Skip this step if your VMs already exist; just make sure the inventory
> hostnames match the OpenStack instance names exactly.

### 2 — (✅) Fresh inventory + collect OpenStack IDs

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8
source ksenv/bin/activate && cd kubespray

cp -rfp inventory/sample inventory/mycluster   # ✅ gives you group_vars/all/openstack.yml

# 📖 from a host with the OpenStack CLI + sourced rc — note the IDs you'll paste below:
source ~/openstack-rc
openstack network list                         # → floating/external network ID, LBaaS network ID
openstack subnet  list                         # → floating subnet, LBaaS VIP subnet, member subnet
openstack catalog list | grep -i load-balancer # confirm Octavia is present
openstack application credential create kubespray-ccm   # → id + secret (shown once!)
```

### 3 — (✅) Enable the external cloud provider — `group_vars/all/all.yml`

```yaml
# inventory/mycluster/group_vars/all/all.yml
cloud_provider: external
external_cloud_provider: openstack
```

> In-tree providers are gone since Kubernetes **v1.31 / Kubespray v2.27** —
> `external` is the only supported value. These two lines are what cause
> `cluster.yml` to deploy the OpenStack cloud-controller-manager.

### 4 — (✅) Auth + Octavia LBaaS — `group_vars/all/openstack.yml`

```yaml
# inventory/mycluster/group_vars/all/openstack.yml

## --- Keystone application credentials (all THREE required; take precedence
##     over OS_USERNAME/OS_PASSWORD from the sourced rc) ---
external_openstack_auth_url: "https://keystone.example.com:5000/v3"
external_openstack_region: "RegionOne"
external_openstack_application_credential_name: "kubespray-ccm"
external_openstack_application_credential_id: "<app-cred-id>"
external_openstack_application_credential_secret: "<app-cred-secret>"

## --- Octavia / LBaaS: gives type: LoadBalancer Services a floating IP ---
external_openstack_lbaas_enabled: true
external_openstack_lbaas_provider: amphora            # or: ovn
external_openstack_lbaas_method: ROUND_ROBIN
external_openstack_lbaas_floating_network_id: "<external/floating network ID>"
external_openstack_lbaas_floating_subnet_id: "<floating subnet ID>"
external_openstack_lbaas_network_id: "<network ID to create the LBaaS VIP>"
external_openstack_lbaas_subnet_id: "<subnet ID to create the LBaaS VIP>"
external_openstack_lbaas_member_subnet_id: "<subnet ID for LB members>"
external_openstack_lbaas_manage_security_groups: false
external_openstack_lbaas_create_monitor: false
external_openstack_lbaas_internal_lb: false

## --- multi-NIC / metadata helpers (leave defaults unless you have several NICs) ---
external_openstack_metadata_search_order: "configDrive,metadataService"
# external_openstack_network_internal_networks: []
# external_openstack_network_public_networks: []
```

> `external_openstack_lbaas_floating_network_id` is the one that hands a
> `LoadBalancer` Service its public floating IP. `provider: amphora` is the
> default Octavia driver; switch to `ovn` only if your cloud runs the OVN
> provider. (Migrating from the old in-tree provider? Replace any
> `openstack_lbaas_subnet_id` with `external_openstack_lbaas_subnet_id`.)

### 5 — (✅) Enable Cinder CSI + a default StorageClass — same file

```yaml
# inventory/mycluster/group_vars/all/openstack.yml  (continued)

cinder_csi_enabled: true
cinder_csi_controller_replicas: 1
# cinder_topology: true          # optional: schedule by volume AZ
# cinder_csi_ignore_volume_az: true

storage_classes:
  - name: "cinder-csi"
    is_default: true
    provisioner: "cinder.csi.openstack.org"
    reclaim_policy: "Delete"
    volume_binding_mode: "WaitForFirstConsumer"
    allow_volume_expansion: true
    mount_options:
      - "discard"
    parameters:
      type: "thin"
      availability: "nova"
```

```yaml
# inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
persistent_volumes_enabled: true   # deploys the cinder provisioner alongside the CSI driver
```

> Cinder CSI is independent of the cloud-controller — it can run on its own — but
> here we enable both. The default class provisioner is
> **`cinder.csi.openstack.org`**. Only **Cinder v3** is supported.

### 6 — (✅ validated) Syntax-check before touching the cloud

```bash
ansible-playbook -i inventory/mycluster/inventory.ini \
  --syntax-check playbooks/cluster.yml          # exit 0 = YAML + vars parse OK
```

### 7 — (📖/🖥️) Source the rc and (re-)deploy

```bash
# 📖 the cloud-controller reads standard OS_* env vars at deploy time:
source ~/openstack-rc

# 🖥️ rolls out the OpenStack CCM + Cinder CSI (re-run on an existing cluster is safe):
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa playbooks/cluster.yml
```

> **Calico / kube-router (L3 CNIs):** OpenStack will drop pod traffic as
> "spoofed" unless you add **allowed-address pairs** on the instance ports for
> both `kube_service_addresses` (`10.233.0.0/18`) and `kube_pods_subnet`
> (`10.233.64.0/18`). Do this on the Neutron ports before expecting Services to
> work.

### 8 — (🖥️) Verify the cloud-controller + cinder-csi Pods

```bash
kubectl -n kube-system get pods | grep -E 'cloud-controller|cinder'
```

### 9 — (🖥️) A LoadBalancer Service gets a floating IP (Octavia)

```bash
kubectl create deployment web --image=nginx --port=80
kubectl expose deployment web --type=LoadBalancer --port=80
kubectl get svc web -w        # watch EXTERNAL-IP go from <pending> to a floating IP
```

### 10 — (🖥️) A PVC provisions a Cinder volume

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc-cinder
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: cinder-csi
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-cinder
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /var/lib/www/html
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: csi-pvc-cinder
EOF

kubectl get pvc csi-pvc-cinder        # WaitForFirstConsumer → Bound once the Pod schedules
kubectl exec -it nginx-cinder -- df -h | grep /var/lib/www/html
```

## Verification

**✅ Validated on the control node (no cloud needed):**

```text
$ ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml
playbook: playbooks/cluster.yml          # exit 0 — openstack.yml + all.yml parse OK
```

**🖥️ Expected against the live cluster (described, not validated):**

```text
$ kubectl -n kube-system get pods | grep -E 'cloud-controller|cinder'
openstack-cloud-controller-manager-xxxxx   1/1   Running
csi-cinder-controllerplugin-xxxxxxxxx      6/6   Running
csi-cinder-nodeplugin-xxxxx                3/3   Running     # one per node

$ kubectl get storageclass
cinder-csi (default)   cinder.csi.openstack.org   Delete   WaitForFirstConsumer   true

$ kubectl get svc web
NAME   TYPE           CLUSTER-IP      EXTERNAL-IP        PORT(S)
web    LoadBalancer   10.233.x.y      <floating-IP>      80:3xxxx/TCP   # IP from the floating network

$ kubectl get pvc csi-pvc-cinder
csi-pvc-cinder   Bound   pvc-xxxx   1Gi   RWO   cinder-csi

$ kubectl exec -it nginx-cinder -- df -h | grep /var/lib/www/html
/dev/vdb   976M   2.6M   958M   1%   /var/lib/www/html      # Cinder volume mounted
```

Pass criteria: `openstack-cloud-controller-manager` and the `csi-cinder-*` Pods
are `Running`; `kubectl get storageclass` shows `cinder-csi` as default with
provisioner `cinder.csi.openstack.org`; the `LoadBalancer` Service's
`EXTERNAL-IP` is populated from the floating network; and the PVC reaches
`Bound` with the volume mounted in the Pod.

📖 Cross-check on the OpenStack side: `openstack loadbalancer list` shows an
amphora LB for the Service, and `openstack volume list` shows a 1 GiB volume
named after the PVC.

## Other clouds (📖, brief)

Same external-CCM + CSI pattern, just a different `external_cloud_provider` and
credentials file under `group_vars/all/`:

| Cloud | Set `external_cloud_provider:` | Creds file | CSI toggle |
|-------|-------------------------------|------------|------------|
| vSphere | `vsphere` | `vsphere.yml` (`external_vsphere_*`, needs 6.7 U3+, HW v15+, VM UUID on) | `vsphere_csi_enabled: true` |
| AWS | (external CCM + EBS CSI) | `aws.yml` | per-cloud EBS CSI |
| Azure / GCP / OCI | (external CCM + cloud CSI) | `azure.yml` / `gcp.yml` | per-cloud CSI |

## Cleanup

```bash
# 🖥️ on the cluster: remove the lab objects (frees the Octavia LB + Cinder volume)
kubectl delete pod nginx-cinder
kubectl delete pvc csi-pvc-cinder
kubectl delete svc web
kubectl delete deployment web

# 📖 OpenStack side, if you created an app credential just for the lab:
openstack application credential delete kubespray-ccm

# 📖 if you provisioned VMs with Terraform and want them gone:
cd contrib/terraform/openstack && terraform destroy -var-file=cluster.tfvars

# Control node: nothing to revert (only YAML was edited under inventory/mycluster).
```
