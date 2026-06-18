# Lesson 5 — Container Runtimes & CNI — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Pick and configure the **Container Runtime Interface (CRI)** that the kubelet
talks to, entirely through `group_vars`:

- select the runtime with **`container_manager`** — `containerd` (default) vs `crio`;
- tune **containerd** — registry mirrors (`containerd_registries_mirrors`) and
  the cgroup driver (`SystemdCgroup`);
- reference an upstream **CRI-O CI scenario** as a known-good config;
- mention the sandboxed runtimes (**gVisor `runsc`** / **Kata `kata-qemu`**)
  layered on containerd and selected **per-Pod** via **RuntimeClass**;
- after a deploy, verify the live runtime on a node with `crictl` and run a
  RuntimeClass-using Pod.

## Prerequisites

- Control node set up (`setup.md`): `python3 -m venv ksenv`, `pip install -r requirements.txt`.
- Inventory copied: `cp -rfp inventory/sample inventory/mycluster` (Lesson 2).
- For the deploy: a reachable fleet (Vagrant or 3 VMs).

The runtime knobs live in **`group_vars/all/`** (not `k8s_cluster/`):
`inventory/mycluster/group_vars/all/containerd.yml` and
`inventory/mycluster/group_vars/all/cri-o.yml` (both come from the sample copy).
`container_manager` is set in `group_vars/k8s_cluster/k8s-cluster.yml`.

## Steps (énoncé)

1. Confirm the default: `container_manager: containerd`.
2. Configure containerd — registry mirror + `SystemdCgroup`, and add an
   insecure/private registry mirror.
3. Validate the edited group_vars are well-formed YAML.
4. (Alternative) Switch to **CRI-O** using the upstream CI scenario as a base.
5. (🖥️) Deploy with `cluster.yml`.
6. (🖥️) Verify the live runtime on a node with `crictl info` / `crictl ps`.
7. (🖥️) Enable a sandboxed runtime (gVisor) and run a **RuntimeClass** Pod.

## Solution

### 1 — `container_manager` (the one switch) — ✅

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoid the locale error
source ksenv/bin/activate && cd kubespray

# containerd is the default; this line lives in k8s-cluster.yml
grep -n container_manager inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
# container_manager: containerd        # ← default; set to 'crio' for CRI-O
```

> gVisor and Kata are **not** `container_manager` values — they ride on top of
> containerd and are picked per-Pod via RuntimeClass (step 7).

### 2 — containerd: mirrors + `SystemdCgroup` — ✅ edit

Edit `inventory/mycluster/group_vars/all/containerd.yml`:

```yaml
# group_vars/all/containerd.yml
# Cgroup driver — MUST match the kubelet's. Use "true" on systemd / cgroups v2.
containerd_runc_runtime:
  name: runc
  type: "io.containerd.runc.v2"
  options:
    SystemdCgroup: "true"

# Registry mirrors (replaces deprecated containerd_registries /
# containerd_insecure_registries). Each entry: a matched `prefix` + a list of
# `mirrors`; an insecure HTTP registry is just a mirror with http:// + skip_verify.
containerd_registries_mirrors:
  - prefix: docker.io
    mirrors:
      - host: https://mirror.gcr.io
        capabilities: ["pull", "resolve"]
        skip_verify: false
    server: https://registry-1.docker.io
  - prefix: 172.19.16.11:5000          # private/insecure registry
    mirrors:
      - host: http://172.19.16.11:5000
        capabilities: ["pull", "resolve"]
        skip_verify: true
```

### 3 — YAML validity — ✅ validated

```bash
# Quick well-formedness check on the edited group_vars (PyYAML ships with Ansible):
python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1])); print('YAML OK')" \
  inventory/mycluster/group_vars/all/containerd.yml
# YAML OK
```

You can also dry-validate the whole inventory render before any deploy:

```bash
ansible-inventory -i inventory/mycluster/inventory.ini --list >/dev/null && echo "inventory OK"
```

### 4 — Alternative: CRI-O from a CI scenario — ✅ edit

The fastest known-good CRI-O base is an upstream CI scenario file. Copy one in
as your config and adapt:

```bash
# Reference configs (real paths in kubernetes-sigs/kubespray, master):
#   tests/files/fedora43-crio.yml
#   tests/files/almalinux9-crio.yml
cp tests/files/almalinux9-crio.yml \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml   # then adapt to your OS/IPs
```

Enabling CRI-O also needs a few keys in `group_vars/all/all.yml`; the registry
config lives in `group_vars/all/cri-o.yml`:

```yaml
# group_vars/all/all.yml
container_manager: crio
download_container: false
skip_downloads: false
etcd_deployment_type: host        # containerd-style etcd-on-host

# group_vars/all/cri-o.yml
crio_registries:
  - prefix: docker.io
    insecure: false
    blocked: false
    location: registry-1.docker.io
    unqualified: false
    mirrors:
      - location: mirror.gcr.io
        insecure: false
# crio_insecure_registries: ["10.0.0.2:5000"]   # HTTP registries
```

```bash
# Same well-formedness check on the CRI-O files:
python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1])); print('YAML OK')" \
  inventory/mycluster/group_vars/all/cri-o.yml
```

### 5 — Deploy — 🖥️ (needs the fleet; ~15-25 min)

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa playbooks/cluster.yml
```

### 6 — Verify the live runtime on a node — 🖥️

`crictl` ships with the node and reads `/etc/crictl.yaml`. SSH to any
`kube_node` and:

```bash
# Which runtime is actually serving the kubelet?
sudo crictl info | grep -iE 'runtimeType|runtimeName|cgroupDriver|SystemdCgroup'
#   containerd → "RuntimeName": "containerd",  "SystemdCgroup": true
#   CRI-O      → "RuntimeName": "crio"

# Sockets confirm the choice:
ls -l /run/containerd/containerd.sock   # containerd
ls -l /var/run/crio/crio.sock           # CRI-O

# Running containers / pod sandboxes through the CRI:
sudo crictl ps
sudo crictl pods
sudo crictl images | head               # mirrors are used at pull time
```

### 7 — Sandboxed runtime + RuntimeClass — 🖥️

Enable gVisor (or Kata) on top of containerd in `group_vars/k8s_cluster/k8s-cluster.yml`,
then re-run `cluster.yml`:

```yaml
container_manager: containerd
gvisor_enabled: true            # OCI runtime 'runsc'; creates RuntimeClass 'gvisor'
# kata_containers_enabled: true # alternative: micro-VM isolation; RuntimeClass 'kata-qemu'
```

Kubespray creates the matching **RuntimeClass** object. A Pod opts in with
`runtimeClassName` (no field → default `runc`):

```yaml
# gvisor-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gvisor-test
spec:
  runtimeClassName: gvisor        # or: kata-qemu
  containers:
  - name: nginx
    image: nginx:1.27
```

```bash
kubectl get runtimeclass                       # gvisor (and/or kata-qemu) present
kubectl apply -f gvisor-test.yaml
kubectl get pod gvisor-test -o wide            # should reach Running
# Prove the sandbox is real (gVisor kernel ≠ host kernel):
kubectl exec gvisor-test -- dmesg | head -1    # shows 'gVisor', not the host
kubectl exec gvisor-test -- uname -a           # gVisor's synthetic kernel string
```

## Verification

**✅ Validated on the control node (no fleet needed):**

```text
$ python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))..." .../containerd.yml
YAML OK              # containerd group_vars (mirrors + SystemdCgroup) parse
YAML OK              # cri-o group_vars parse
$ ansible-inventory -i inventory/mycluster/inventory.ini --list >/dev/null
inventory OK
```

**🖥️ After deploy:**

- `crictl info` reports the expected `RuntimeName` (`containerd` or `crio`) and
  `SystemdCgroup: true` when you set it; the matching socket exists.
- `crictl ps` / `crictl pods` list the running CRI workloads on the node.
- `kubectl get runtimeclass` shows `gvisor` (and/or `kata-qemu`); the
  `gvisor-test` Pod is `Running` and `dmesg`/`uname` inside it report the
  sandbox kernel, not the host's.

## Faster lab variant — CRI-O all-in-one

To exercise CRI-O on a single node, start from a CI scenario and point the
inventory at one host in all three groups:

```bash
cp tests/files/almalinux9-crio.yml \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
# ensure container_manager: crio + the all.yml keys from step 4, then run cluster.yml
```

## Cleanup

```bash
kubectl delete -f gvisor-test.yaml --ignore-not-found
ansible-playbook -i inventory/mycluster/inventory.ini -b \
  -e reset_confirmation=yes playbooks/reset.yml
```

---

## Objective

Pick the cluster CNI with the single `kube_network_plugin` switch, then tune the
network: set Calico encapsulation (`calico_vxlan_mode` / `calico_ipip_mode`) and
the pod/service CIDRs (`kube_pods_subnet`, `kube_service_addresses`) in
`group_vars/k8s_cluster/`. Use a CI scenario as a known-good base, validate the
YAML on the control node, then (🖥️) deploy and prove pod-to-pod connectivity with
**netchecker** and a cross-node Pod ping.

## Prerequisites

- Control node set up (`setup.md`): venv + `pip install -r requirements.txt`.
- `inventory/mycluster` created (`cp -rfp inventory/sample inventory/mycluster`),
  with a reachable fleet in `inventory/mycluster/inventory.ini` for the 🖥️ steps.

## Steps (énoncé)

1. Choose a CNI by copying a matching CI scenario into your inventory and
   confirming `kube_network_plugin`.
2. Set the pod and service CIDRs in `k8s-cluster.yml` (non-overlapping ranges).
3. Tune Calico encapsulation in `k8s-net-calico.yml`
   (`calico_vxlan_mode` / `calico_ipip_mode`).
4. Turn on the connectivity self-test: `deploy_netchecker: true`.
5. Validate the edited YAML + `cluster.yml` syntax on the control node.
6. (🖥️) Deploy, then query netchecker and run a cross-node Pod ping.

## Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoid the locale error
source ksenv/bin/activate && cd kubespray

GV=inventory/mycluster/group_vars/k8s_cluster
```

### 1 — Choose a CNI from a CI scenario (✅ pick one)

The `tests/files/*.yml` files are complete, CI-validated configs. Use one as the
base for `k8s-cluster.yml`, then adapt:

```bash
# Calico (the default). NOTE: almalinux9-calico.yml does NOT set
# kube_network_plugin at all — Calico is the default, so omitting it selects it.
cp tests/files/almalinux9-calico.yml      $GV/k8s-cluster.yml

# …or Cilium (eBPF, kube-proxy replacement):
#   cp tests/files/rockylinux10-cilium.yml  $GV/k8s-cluster.yml
#   → sets  kube_network_plugin: cilium  and  cilium_kube_proxy_replacement: true

# …or Flannel (simple VXLAN, HA control plane):
#   cp tests/files/ubuntu24-flannel-ha.yml  $GV/k8s-cluster.yml
#   → sets  kube_network_plugin: flannel
```

> These scenarios carry CI-only instance keys (`cloud_image`, `vm_memory`,
> `mode`) — harmless leftovers; the lines that matter are `kube_network_plugin`
> and the tuning below. For Calico, make the choice explicit rather than relying
> on the default:

```bash
grep -q '^kube_network_plugin:' $GV/k8s-cluster.yml \
  || echo 'kube_network_plugin: calico' >> $GV/k8s-cluster.yml
```

### 2 — Pod & Service CIDRs (✅ edit `k8s-cluster.yml`)

Pick ranges that don't overlap your infra network. In `$GV/k8s-cluster.yml`:

```yaml
kube_service_addresses: 10.233.0.0/18      # Service ClusterIP range
kube_pods_subnet: 10.233.64.0/18           # Pod IP range
kube_network_node_prefix: 24               # per-node pod block carved from above
```

```bash
# idempotent set (adds if missing, replaces if present)
for kv in \
  'kube_service_addresses: 10.233.0.0/18' \
  'kube_pods_subnet: 10.233.64.0/18' \
  'kube_network_node_prefix: 24'; do
  key=${kv%%:*}
  grep -q "^$key:" $GV/k8s-cluster.yml \
    && sed -i "s|^$key:.*|$kv|" $GV/k8s-cluster.yml \
    || echo "$kv" >> $GV/k8s-cluster.yml
done
```

### 3 — Calico encapsulation (✅ edit `k8s-net-calico.yml`)

Encapsulation is set independently per overlay. VXLAN is the more mature path and
is on by default; IPIP requires the BIRD backend. In `$GV/k8s-net-calico.yml`:

```yaml
# VXLAN only between subnets (no encap inside a subnet) — common, efficient
calico_vxlan_mode: CrossSubnet            # Always (default) | CrossSubnet | Never
calico_ipip_mode: Never                   # Never (default) | Always | CrossSubnet
# calico_network_backend: vxlan           # default; use 'bird' for BGP/IPIP
```

```bash
for kv in \
  'calico_vxlan_mode: CrossSubnet' \
  'calico_ipip_mode: Never'; do
  key=${kv%%:*}
  grep -q "^$key:" $GV/k8s-net-calico.yml \
    && sed -i "s|^$key:.*|$kv|" $GV/k8s-net-calico.yml \
    || echo "$kv" >> $GV/k8s-net-calico.yml
done
```

> `calico_vxlan_mode` / `calico_ipip_mode` live in `k8s-net-calico.yml`
> (Calico-only). They are ignored when `kube_network_plugin` is `cilium` or
> `flannel`; those plugins have their own tunables in `k8s-net-cilium.yml` /
> `k8s-net-flannel.yml`.

### 4 — Connectivity self-test (✅ enable netchecker)

```bash
grep -q '^deploy_netchecker:' $GV/k8s-cluster.yml \
  && sed -i 's|^deploy_netchecker:.*|deploy_netchecker: true|' $GV/k8s-cluster.yml \
  || echo 'deploy_netchecker: true' >> $GV/k8s-cluster.yml
```

### 5 — Validate on the control node (✅ no nodes needed)

```bash
# YAML well-formed?  (both files must parse)
python3 -c "import yaml,sys; [yaml.safe_load(open(f)) for f in sys.argv[1:]]; print('YAML OK')" \
  $GV/k8s-cluster.yml $GV/k8s-net-calico.yml

# Confirm the knobs are set
grep -E '^(kube_network_plugin|kube_pods_subnet|kube_service_addresses|deploy_netchecker):' \
  $GV/k8s-cluster.yml
grep -E '^calico_(vxlan|ipip)_mode:' $GV/k8s-net-calico.yml

# Playbook still parses with the edited group_vars
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml
```

### 6 — Deploy + validate connectivity (🖥️ needs the fleet)

```bash
# deploy with the CNI + CIDRs chosen above (~15-25 min)
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa playbooks/cluster.yml

# CNI pods up?
kubectl -n kube-system get pods -l k8s-app=calico-node -o wide   # cilium / kube-flannel for the others

# netchecker overall status (agents report pod-to-pod + DNS reachability)
curl -s http://localhost:31081/api/v1/connectivity_check | python3 -m json.tool

# cross-node Pod ping: schedule two pods, ping one from the other
kubectl run p1 --image=nicolaka/netshoot --command -- sleep 3600
kubectl run p2 --image=nicolaka/netshoot --command -- sleep 3600
kubectl wait --for=condition=Ready pod/p1 pod/p2 --timeout=120s
kubectl get pod p1 p2 -o wide                                    # confirm different NODEs
P2_IP=$(kubectl get pod p2 -o jsonpath='{.status.podIP}')
kubectl exec p1 -- ping -c3 "$P2_IP"
```

## Verification

**✅ Validated on the control node (no fleet needed):**

```text
$ python3 -c "import yaml,sys; [yaml.safe_load(open(f)) for f in sys.argv[1:]]; print('YAML OK')" \
    $GV/k8s-cluster.yml $GV/k8s-net-calico.yml
YAML OK

$ grep -E '^(kube_network_plugin|kube_pods_subnet|kube_service_addresses|deploy_netchecker):' \
    $GV/k8s-cluster.yml
kube_network_plugin: calico
kube_pods_subnet: 10.233.64.0/18
kube_service_addresses: 10.233.0.0/18
deploy_netchecker: true

$ grep -E '^calico_(vxlan|ipip)_mode:' $GV/k8s-net-calico.yml
calico_vxlan_mode: CrossSubnet
calico_ipip_mode: Never

$ ansible-playbook ... --syntax-check playbooks/cluster.yml
playbook: playbooks/cluster.yml          # exit 0 — syntax OK
```

**🖥️ After deploy:** the CNI Pods (`calico-node` / `cilium` / `kube-flannel`)
are `Running` on every node; the netchecker API reports
`"Message": "Connectivity to all nodes is OK"` (all agents reachable); and the
cross-node `ping p1 → p2` shows `0% packet loss`, proving the overlay carries
pod-to-pod traffic between different nodes.

## Faster lab variant — single node, Calico

For a quick CNI smoke test on one node, base `k8s-cluster.yml` on the all-in-one
scenario and keep Calico's default VXLAN:

```bash
cp tests/files/ubuntu22-calico-all-in-one.yml \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
# put one node in all of [kube_control_plane] [etcd] [kube_node], then run cluster.yml
```

> A single node can't exercise *cross-node* traffic, but netchecker and a
> two-Pod same-node ping still validate the CNI datapath and DNS.

## Cleanup

```bash
# delete the test pods
kubectl delete pod p1 p2 --ignore-not-found

# tear down the cluster
ansible-playbook -i inventory/mycluster/inventory.ini -b \
  -e reset_confirmation=yes playbooks/reset.yml
```
