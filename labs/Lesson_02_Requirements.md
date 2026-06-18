# Lesson 2 — Requirements & Environment — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Bring up the **Ansible control node** the way Kubespray pins it (Python venv +
`pip install -r requirements.txt`), then use Ansible **ad-hoc** commands to
verify the **target-node prerequisites** across the whole fleet at once:
supported OS, swap off, IPv4 forwarding + `br_netfilter`, and the required
control-plane / worker ports. This is the preflight you run *before* Lesson 6's
`cluster.yml`.

## Prerequisites

- A control node (your workstation / bastion) with `python3` and `git`.
  > ⚠️ **Match Kubespray to your Python.** Kubespray **master** pins
  > `ansible==11.x` (**ansible-core 2.18**, needs **Python 3.11–3.13**). On a
  > default **Ubuntu 22.04** control node (Python **3.10**) that `pip install`
  > fails — use a release branch that targets 2.16, e.g. **`release-2.27`**
  > (`ansible==9.13`, ansible-core **2.16**, K8s v1.31.x), which is what this lab
  > cluster was built with.
- The Kubespray repo checked out: `git clone https://github.com/kubernetes-sigs/kubespray`.
- (For the 🖥️ steps only) a reachable fleet in
  `inventory/mycluster/inventory.ini` and an SSH key that works on every node.
- 📖 Reference docs:
  [Getting started](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting_started/getting-started.md) ·
  [Kernel requirements](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/kernel-requirements.md) ·
  [Port requirements](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/port-requirements.md)

## Steps (énoncé)

1. (✅) Create and activate the control-node Python venv.
2. (✅) Install Ansible exactly as Kubespray pins it (`requirements.txt`), then
   confirm the `ansible` / `python3` versions.
3. (✅) Sanity-check that `python-netaddr` came along (Ansible needs it for IPs).
4. (🖥️) Confirm every target node is reachable over SSH with `become`.
5. (🖥️) Check the **OS** is a supported distribution on each node.
6. (🖥️) Check **swap is off** on every node.
7. (🖥️) Check **IPv4 forwarding** (`net.ipv4.ip_forward = 1`) and the
   `br_netfilter` module on every node.
8. (🖥️) Confirm the required **ports** are free/listening on the right nodes
   (6443/2379/2380/10250).

## Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoid the Ansible locale warning

# 1 — control-node venv (✅)
VENVDIR=ksenv
python3 -m venv $VENVDIR
source $VENVDIR/bin/activate
cd kubespray

# 2 — install Ansible as Kubespray pins it, then check versions (✅)
pip install -U pip
pip install -r requirements.txt              # pulls ansible-core 2.18.x, jinja, netaddr
ansible --version                            # expect core within 2.18.x
python3 --version                            # 3.11 - 3.13 for Ansible 2.18.x

# 3 — netaddr present? (✅) — Ansible's ipaddr filters need it
python3 -c "import netaddr; print('netaddr', netaddr.__version__)"

# ---- everything below is 🖥️ : needs the VM fleet ----

# 4 — reachability + privilege escalation
ansible all -i inventory/mycluster/inventory.ini -m ping -b

# 5 — supported OS on each node (distro + version)
ansible all -i inventory/mycluster/inventory.ini -m setup \
  -a 'filter=ansible_distribution*'
#   or quick-and-dirty:
ansible all -i inventory/mycluster/inventory.ini -m shell -a 'cat /etc/os-release | head -2'

# 6 — swap must be OFF (no output = good)
ansible all -i inventory/mycluster/inventory.ini -m shell -a 'swapon --show'

# 7 — IPv4 forwarding (must be 1) and br_netfilter module
ansible all -i inventory/mycluster/inventory.ini -m shell -a 'cat /proc/sys/net/ipv4/ip_forward'
ansible all -i inventory/mycluster/inventory.ini -m shell -a 'lsmod | grep br_netfilter || echo MISSING'

# 8 — ports. Pre-deploy these should be FREE; post-deploy they should LISTEN.
#     Control-plane nodes: API 6443, etcd 2379/2380, kubelet 10250
ansible kube_control_plane -i inventory/mycluster/inventory.ini -b \
  -m shell -a "ss -tlnp | grep -E ':(6443|2379|2380|10250)' || echo 'none listening (expected pre-deploy)'"
#     Worker nodes: kubelet 10250 + NodePort range 30000-32767
ansible kube_node -i inventory/mycluster/inventory.ini -b \
  -m shell -a "ss -tlnp | grep -E ':10250' || echo 'none listening (expected pre-deploy)'"
```

If a node fails the IPv4-forwarding check, fix it on that node (persisted):

```bash
# 🖥️ remediation, run per node or via ansible -m shell -b
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-k8s.conf
sudo sysctl --system
```

> Kubespray itself loads `br_netfilter`/`overlay`, sets the bridge sysctls and
> turns swap off during `preinstall`; step 6–7 are a *preflight* so you catch a
> locked-down image before the deploy, not a replacement for the role.

## Verification

**✅ Validated on the control node (no fleet needed):**

```text
$ ansible --version
ansible [core 2.18.x]
  python version = 3.1x.x

$ python3 -c "import netaddr; print('netaddr', netaddr.__version__)"
netaddr 1.x.x
```

**🖥️ Expected against the fleet (described, not validated):**

```text
$ ansible all ... -m ping -b
node1 | SUCCESS => { "ping": "pong" }        # one line per node, all SUCCESS

$ ansible all ... -m shell -a 'swapon --show'
node1 | CHANGED | rc=0 >>                      # empty body = swap OFF (good)

$ ansible all ... -m shell -a 'cat /proc/sys/net/ipv4/ip_forward'
node1 | CHANGED | rc=0 >>
1                                              # 1 = forwarding ON (good)

$ ansible all ... -m shell -a 'lsmod | grep br_netfilter || echo MISSING'
node1 | CHANGED | rc=0 >>
br_netfilter           32768  0                # present (or "MISSING" before deploy)

$ ansible kube_control_plane ... ss -tlnp | grep -E ':(6443|2379|2380|10250)'
none listening (expected pre-deploy)           # BEFORE cluster.yml
# AFTER cluster.yml the same command shows kube-apiserver:6443, etcd:2379/2380,
# kubelet:10250 all LISTEN.
```

Pass criteria: OS is on the
[supported list](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting_started/getting-started.md),
`swapon --show` is empty, `ip_forward` is `1`, and no foreign process already
owns 6443/2379/2380/10250 before you deploy.

## Cleanup

```bash
# Control node: drop the venv (✅)
deactivate
rm -rf ksenv

# Fleet: nothing was changed except the optional sysctl remediation. To revert: (🖥️)
ansible all -i inventory/mycluster/inventory.ini -b \
  -m file -a 'path=/etc/sysctl.d/99-k8s.conf state=absent'
```
