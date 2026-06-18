# Kubernetes Production (CKA + Kubespray) — Lab Workbook (Student Questions)

These labs all run against **one shared, multi-node Kubernetes cluster deployed on OpenStack via Kubespray** — you build it yourself in **Lab 0** (see `setup.md`), and every later lab assumes it is up and `Ready`. Some labs are *infra* labs marked 🖥️: they are driven from the **Ansible control node** against `inventory/lab/` (inventory and `group_vars` edits, `cluster.yml`, `scale.yml`, `remove-node.yml`, `upgrade-cluster.yml`, `recover-control-plane.yml`, …). The rest are *usage* labs run with `kubectl` against the cluster's `admin.conf` — identical to any cluster, but here the OpenStack **Octavia** load balancer and **Cinder CSI** back the Access and Storage labs. Throughout, **substitute the real node names from `kubectl get nodes`** wherever you see `cka-lab-*`, `node1`–`node6`, or similar placeholders. The énoncé tells you *what* to accomplish; figuring out the *how* is the point. Verify your own work with `kubectl get`/`describe` (or the `> Verify:` line) before moving on, and clean up your namespaces when you finish.

## Summary

| Lab | Estimated time | Notes |
|-----|----------------|-------|
| 0 — Deploy the cluster on OpenStack via Kubespray | 90 min | 🖥️ infra · foundation |
| 1 — Kubernetes Architecture | 10 min | |
| 2 — Requirements & Environment | 20 min | |
| 3 — Installation & Inventory | 25 min | |
| 4 — Cluster Configuration (group_vars) | 30 min | |
| 5 — Container Runtimes & CNI | 25 min | |
| 6 — Deploying the Cluster (`cluster.yml`) | 45 min | 🖥️ long run |
| 7 — High Availability (failover) | 30 min | 🖥️ |
| 8 — Deploying Applications & Autoscaling | 25 min | |
| 9 — Managing Storage (Cinder) | 15 min | |
| 10 — Application Access (Octavia/Ingress/Gateway) | 25 min | |
| 11 — Configuration & Quotas | 15 min | |
| 12 — Scheduling | 20 min | |
| 13 — Networking & NetworkPolicy | 20 min | |
| 14 — Security (RBAC) | 20 min | |
| 15 — CRDs & Operators | 15 min | |
| 16 — Node Maintenance & Scaling | 35 min | 🖥️ |
| 17 — Upgrades (`upgrade-cluster.yml`) | 40 min | 🖥️ long run |
| 18 — Backup, Recovery & Reset | 35 min | 🖥️ |
| 19 — Add-ons & Packaging | 25 min | |
| 20 — Troubleshooting | 20 min | |
| 21 — Hardening & Offline | 20 min | |
| 22 — Cloud & OpenStack | 25 min | 🖥️ |
| 23 — Practice Exam | 60 min | |
| **TOTAL** | **690 min** | |

---

## Lab 0 — Deploy the cluster on OpenStack via Kubespray

**⏱ 90 min**   🖥️ *(infra · foundation — run from the Ansible control node)*

**Objective:** Stand up the **shared lab cluster** that every other lab in this workbook uses: a multi-node Kubernetes cluster on OpenStack VMs, provisioned by Kubespray, with the OpenStack integration (external cloud provider + **Octavia** LBaaS + **Cinder CSI**) wired in so `LoadBalancer` Services and dynamic PVCs work. Full details and reference commands live in `setup.md`.

> Prerequisites: an OpenStack tenant with quota for ~6 instances (~12 vCPU / ~24 GB RAM), ≥5 floating IPs, one Octavia LB and Cinder volume quota; **application credentials** (preferred) or an `openrc`; an Ubuntu/Debian/Rocky image and a flavor (≥ 2 vCPU / 4 GB for control planes); and an SSH keypair imported into OpenStack.

### Tasks

1. Prepare the **Ansible control node**: clone Kubespray, create a Python venv, and `pip install -r requirements.txt`.
2. Provision the VM fleet (3 control-plane/etcd + 2 workers). Either use **Path A** — `contrib/terraform/openstack` to create the instances/networks/security-groups *and* generate the dynamic inventory — or **Path B** — create the VMs by hand and write `inventory/lab/inventory.ini` placing nodes in `[kube_control_plane]`, `[etcd]`, `[kube_node]`, `[k8s_cluster:children]`.
3. Configure the OpenStack cloud integration in `group_vars`: in `all/all.yml` set `cloud_provider: external`, `external_cloud_provider: openstack`, `cinder_csi_enabled: true`, `persistent_volumes_enabled: true`; in `all/openstack.yml` set the Keystone auth + Octavia values (`external_openstack_auth_url`, application credential id/secret, region, `external_openstack_lbaas_subnet_id`, `external_openstack_lbaas_floating_network_id`).
4. Deploy with `ansible-playbook -i inventory/lab/hosts -b cluster.yml` (~20–40 min), then point your `KUBECONFIG` at `inventory/lab/artifacts/admin.conf`.
5. Verify the baseline every later lab assumes: `kubectl get nodes` shows all nodes `Ready`; a `type: LoadBalancer` Service receives a floating IP from Octavia; the `cinder-csi` StorageClass provisions a Cinder volume for a PVC.

> Verify: all nodes (3 control-plane + 2 workers) are `Ready`; a test `LoadBalancer` Service gets an `EXTERNAL-IP` (Octavia floating IP); `kubectl get sc` shows `cinder-csi` (`cinder.csi.openstack.org`) provisioning a Cinder volume for a PVC.

---

## Lab 1 — Kubernetes Architecture

**⏱ 10 min**

**Objective:** Map the running cluster against the control-plane / worker architecture in your head.

### Tasks

1. Find how many nodes there are, and which one is the control plane.
2. List the Pods that make up the control plane.
3. Find one DaemonSet that runs on every node, and explain why it must run everywhere.
4. Find the IP address (host:port) of the API-server endpoint.
5. Find which node is currently running the `metrics-server` Pod (if installed).

> Verify: step 2 should yield exactly four Pods (one each of `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`); the DaemonSets you find (`kube-proxy` and the CNI) should have one Pod per node.

---

## Lab 2 — Requirements & Environment

**⏱ 20 min**

**Objective:** Bring up the **Ansible control node** the way Kubespray pins it (Python venv + `pip install -r requirements.txt`), then use Ansible **ad-hoc** commands to verify the **target-node prerequisites** across the whole fleet at once — supported OS, swap off, IPv4 forwarding + `br_netfilter`, and the required ports. This is the preflight you run *before* Lab 6's `cluster.yml`.

### Tasks

1. Create and activate the control-node Python venv.
2. Install Ansible exactly as Kubespray pins it (`requirements.txt`), then confirm the `ansible` / `python3` versions.
3. Sanity-check that `python-netaddr` came along (Ansible needs it for IP filters).
4. 🖥️ Confirm every target node is reachable over SSH with `become` (`ansible all -m ping -b`).
5. 🖥️ Check the **OS** is a supported distribution on each node.
6. 🖥️ Check **swap is off** on every node.
7. 🖥️ Check **IPv4 forwarding** (`net.ipv4.ip_forward = 1`) and the `br_netfilter` module on every node.
8. 🖥️ Confirm the required **ports** are free/listening on the right nodes (6443/2379/2380/10250 on control planes; 10250 on workers).

> Verify: `ansible --version` reports core 2.18.x and `netaddr` imports; against the fleet, every node pings `SUCCESS`, the OS is on the supported list, `swapon --show` is empty, `ip_forward` is `1`, `br_netfilter` is present, and no foreign process owns 6443/2379/2380/10250 before you deploy.

---

## Lab 3 — Installation & Inventory

**⏱ 25 min**

**Objective:** Stand up the Kubespray control node and produce a parseable inventory: clone Kubespray into a venv, install its Ansible deps, copy `inventory/sample` → `inventory/mycluster`, edit `inventory.ini` to place nodes in the right groups, then prove the inventory parses (and — when a fleet exists — that Ansible can reach every node).

> Note: the old `contrib/inventory_builder/inventory.py` script has been removed upstream — the inventory is edited **manually** now.

### Tasks

1. Create and activate a Python virtualenv, then clone Kubespray (pin a release tag — never run from a moving `master` in production).
2. `pip install -r requirements.txt` into the venv.
3. Copy the bundled sample inventory to your own `inventory/mycluster`.
4. Edit `inventory.ini`: declare your hosts, then place them in `[kube_control_plane]`, `[etcd]`, `[kube_node]`, and list the two child groups under `[k8s_cluster:children]`.
5. Parse the inventory and confirm the five groups appear (`ansible-inventory --list`).
6. 🖥️ Verify Ansible can SSH and become root on every node (`ansible all -m ping -b`).

> Verify: `ansible-inventory --list` resolves to exactly the five groups `all`, `etcd`, `k8s_cluster`, `kube_control_plane`, `kube_node`, with `k8s_cluster` the derived union of `kube_control_plane` + `kube_node`; the ping step reports `"ping": "pong"` for every host.

---

## Lab 4 — Cluster Configuration (group_vars)

**⏱ 30 min**

**Objective:** Turn a generic copied inventory into *your* cluster by editing the handful of `group_vars` that shape every Kubespray cluster — `k8s-cluster.yml` (version/CNI/runtime/subnets), `addons.yml` (metrics-server/helm), `all.yml` (NTP/proxy/loadbalancer) — then prove the edits are valid YAML and the inventory still parses.

### Tasks

1. (Optional) Seed `k8s-cluster.yml` from a known-good CI scenario (e.g. `tests/files/ubuntu22-calico-all-in-one.yml`) so you start from a config that already deploys, then adapt it.
2. Edit `group_vars/k8s_cluster/k8s-cluster.yml`: set `kube_version`, `kube_network_plugin: calico`, `container_manager: containerd`, and the `kube_service_addresses` / `kube_pods_subnet` CIDRs (non-overlapping).
3. Edit `group_vars/k8s_cluster/addons.yml`: enable `metrics_server_enabled` and `helm_enabled`.
4. Edit `group_vars/all/all.yml`: turn on NTP, (optionally) set an HTTP(S) proxy, and enable the internal API-server loadbalancer.
5. Validate each edited file is still valid YAML.
6. Re-run `ansible-inventory --list` to confirm the inventory still parses to the 5 groups with your `group_vars` applied (vars now show on the hosts).

> Verify: the three edited files all parse as valid YAML and `ansible-inventory --list` still resolves to the 5 groups with your vars carried on the host entries (e.g. `kube_network_plugin: calico`, your `kube_version`).

---

## Lab 5 — Container Runtimes & CNI

**⏱ 25 min**

**Objective:** Two configuration exercises driven entirely from `group_vars`: **(A)** pick and configure the **container runtime** the kubelet talks to (CRI), and **(B)** pick and tune the cluster **CNI**. Validate the YAML on the control node; the deploy/verify steps are 🖥️ and assume the cluster from Lab 6.

### Tasks

**Part A — Container Runtimes (CRI)**

1. Confirm the default: `container_manager: containerd`.
2. Configure containerd — add a registry mirror (incl. an insecure/private registry) and set the `SystemdCgroup` cgroup driver.
3. Validate the edited group_vars are well-formed YAML.
4. (Alternative) Switch to CRI-O using the upstream CI scenario as a base.
5. 🖥️ Deploy with `cluster.yml`, then verify the live runtime on a node with `crictl info` / `crictl ps`.
6. 🖥️ Enable a sandboxed runtime (gVisor `runsc` / Kata) and run a RuntimeClass Pod, confirming it lands in a sandbox.

**Part B — CNI**

7. Choose a CNI by copying a matching CI scenario into your inventory and confirming `kube_network_plugin`.
8. Set the pod and service CIDRs in `k8s-cluster.yml` (non-overlapping ranges).
9. Tune Calico encapsulation in `k8s-net-calico.yml` (`calico_vxlan_mode` / `calico_ipip_mode`).
10. Turn on the connectivity self-test: `deploy_netchecker: true`.
11. Validate the edited YAML + `cluster.yml` syntax on the control node.
12. 🖥️ Deploy, then query netchecker and run a cross-node Pod ping.

> Verify: the runtime and CNI group_vars parse as YAML and `--syntax-check` passes; after deploy `crictl info` reports the expected `RuntimeName` with `SystemdCgroup: true`, `kubectl get runtimeclass` shows the sandbox class with its Pod Running, the CNI Pods run on every node, and netchecker plus a cross-node ping report 0% packet loss.

---

## Lab 6 — Deploying the Cluster (`cluster.yml`)

**⏱ 45 min**   🖥️ *(long run)*

**Objective:** Take a prepared inventory, validate it, run `cluster.yml` to deploy a cluster, retrieve the kubeconfig, and verify the cluster — the full Day-1 flow.

### Tasks

1. Parse the inventory and confirm the groups are right.
2. `--syntax-check` `cluster.yml` before touching the nodes.
3. 🖥️ Deploy with `cluster.yml`.
4. Make the cluster reachable from the control node (retrieve the kubeconfig).
5. Verify nodes and system Pods are healthy.

> Verify: `ansible-inventory --list` shows the expected groups/hosts and `--syntax-check` passes (exit 0); after deploy `kubectl get nodes` shows every node `Ready` and the CNI, CoreDNS, and kube-proxy Pods are Running in `kube-system`.

---

## Lab 7 — High Availability (failover)

**⏱ 30 min**   🖥️

**Objective:** Build an HA control plane — put 3 nodes in `kube_control_plane` **and** `etcd`, choose how the kube-apiserver is load-balanced, choose the `etcd_deployment_type`, deploy, then prove HA by stopping one control-plane node and watching the API stay up.

### Tasks

1. Edit the inventory: 3 nodes in `kube_control_plane` with `etcd` stacked on them (odd count → quorum survives 1 failure), plus worker nodes.
2. Pick **one** API-server LB mode and set it in `group_vars/all/all.yml`: localhost LB (default), external `loadbalancer_apiserver`, or `kube_vip` VIP.
3. Pick `etcd_deployment_type` (`host` default, or `kubeadm` static-Pod) in `group_vars/all/etcd.yml`; for external etcd, move the members to their own nodes.
4. Parse the inventory and `--syntax-check` `cluster.yml`.
5. 🖥️ Deploy with `cluster.yml`.
6. 🖥️ Verify HA: all control-plane Pods Running, `etcdctl member list` shows 3 members, then stop one control-plane node and confirm the API survives.

> Verify: the inventory resolves to 3 `kube_control_plane` / 3 `etcd` and `--syntax-check` passes; after deploy all 3 control-plane nodes are Ready with 3 etcd members, and after stopping one node `kubectl get --raw=/readyz` keeps returning ok (quorum 2/3) while that node goes NotReady, recovering on restart.

---

## Lab 8 — Deploying Applications & Autoscaling

**⏱ 25 min**

**Objective:** Two exercises: **(A)** ship a small web app with a controlled rollout, a DaemonSet and a StatefulSet in one namespace; **(B)** autoscale a Deployment on CPU and observe a real scale-out under load (requires `metrics-server`).

### Tasks

**Part A — Deployment + DaemonSet + StatefulSet**

1. Create namespace `l3-lab`.
2. Create a Deployment `web` with image `nginx:1.27`, 3 replicas, label `tier=frontend` (on both the Deployment and the Pod template).
3. Update the Deployment to `nginx:1.28` and confirm the rollout completes.
4. Scale it up to 5 replicas, then back down to 2.
5. Roll the Deployment back to revision 1.
6. Create a DaemonSet `agent` (busybox sleeping) that runs on every worker node (not the control plane).
7. Create a StatefulSet `data` of 2 replicas, each with its own 50 Mi PVC.
8. Confirm pods `data-0` and `data-1` exist and each has its own PVC.

**Part B — Horizontal Pod Autoscaler**

9. In namespace `hpa-lab`, deploy `web` from `registry.k8s.io/hpa-example` with `requests.cpu=100m`, `limits.cpu=500m`; expose it on port 80.
10. Create an HPA (`autoscaling/v2`) targeting **50%** CPU, **min 2**, **max 8**.
11. Generate sustained load from an in-cluster Pod; watch `kubectl get hpa -w` until `REPLICAS` rises above 2.
12. Stop the load and confirm the HPA reports CPU back under target.

> Verify: after the rollback the Deployment image is back to `nginx:1.27`; 2 DaemonSet Pods (one per worker); 2 StatefulSet Pods + 2 PVCs. Under load the HPA `TARGETS` exceeds 50% and `REPLICAS` scales toward 8; after removing the load, replicas trend back toward 2 after the stabilisation window.

---

## Lab 9 — Managing Storage (Cinder)

**⏱ 15 min**

**Objective:** Use both dynamic (Cinder CSI) and static provisioning, observe binding, and prove data survives a Pod restart.

### Tasks

1. Create namespace `l4-lab`.
2. Create a 200 Mi PVC `app-data` using the default StorageClass.
3. Create a Pod `writer` that writes `Hello-<random>` into `/data/marker` and then sleeps.
4. Delete `writer`. Create a new Pod `reader` referencing the **same** PVC, and confirm `/data/marker` still contains the same value.
5. Create a static PV `archive-pv` of 100 Mi (storageClass `archive`, hostPath `/tmp/archive`, RWO, Retain). Bind a PVC `archive-claim` to it.

> Verify: `app-data` is Bound to a dynamically-created PV (the default Cinder CSI SC), `archive-claim` is Bound to `archive-pv`, and the value `reader` reads matches what `writer` wrote.

---

## Lab 10 — Application Access (Octavia/Ingress/Gateway)

**⏱ 25 min**

**Objective:** Two exercises: **(A)** expose a single Deployment several ways (ClusterIP, NodePort, Ingress) and observe each; **(B)** install the Gateway API CRDs and model HTTP routing with a `GatewayClass` / `Gateway` / `HTTPRoute`.

> Context: the Ingress task needs an ingress controller installed (see the setup section of the source lab); on this cluster a `type: LoadBalancer` Service is backed by Octavia.

### Tasks

**Part A — Ingress / Service exposure**

1. Create namespace `l5-lab`. Deploy `hello` from `hashicorp/http-echo:1.0`, listening on `:5678` with `-text=hello-from-pod` (use the downward API for the Pod name), 2 replicas.
2. Create a ClusterIP Service `hello` exposing port 80 → 5678.
3. Create a NodePort Service `hello-np` for the same Deployment.
4. From a Pod inside the cluster, `curl http://hello/` 5 times and observe round-robin between the two Pods.
5. Create an Ingress `hello-ing` routing `Host: hello.local` to `hello:80` via the `nginx` ingress class.

**Part B — Gateway API**

6. Install the official Gateway API **standard CRDs**.
7. In namespace `gw-lab`, deploy `site` (`hashicorp/http-echo:1.0`, text `routed-by-gateway`, listen `:5678`) and expose it as `site:80`.
8. Create a `GatewayClass` `lab-class`, a `Gateway` `lab-gw` (HTTP listener on `:80`), and an `HTTPRoute` `site-route` routing `Host: site.lab` (prefix `/`) to `site:80`.
9. Show the three resources and explain **why** the `Gateway` has no `ADDRESS`.

> Verify: `hello` is a ClusterIP with 2 endpoints, `hello-np` a NodePort with an assigned port, and a curl through the ingress (header `Host: hello.local`) returns `hello-from-pod`; the 5 Gateway-API CRDs exist and `GatewayClass`/`Gateway`/`HTTPRoute` are created, with the `Gateway` showing no `ADDRESS` and `PROGRAMMED=Unknown` because no controller implements its class.

---

## Lab 11 — Configuration & Quotas

**⏱ 15 min**

**Objective:** Deliver a multi-tenant namespace `dev` with a hard Pod cap, resource quota, sensible per-Container defaults, plus a Pod wired to a ConfigMap and a Secret.

### Tasks

1. Create namespace `dev`.
2. Create a LimitRange so any new Container gets defaultRequest `100m`/`128Mi` and default `500m`/`512Mi`.
3. Create a ResourceQuota: `pods=4`, `requests.cpu=2`, `requests.memory=2Gi`, `limits.cpu=4`, `limits.memory=4Gi`.
4. Create ConfigMap `app-config` with `LOG_LEVEL=debug` and a key `app.properties` containing `feature.x=true` / `feature.y=false`.
5. Create Secret `app-secret` with `api_key=ABCD-1234`.
6. Create Deployment `webapp` (1 replica, `nginx:1.27`) that:
   - sets env `LOG_LEVEL` from the ConfigMap;
   - mounts the `app.properties` key as `/etc/app/app.properties`;
   - mounts the Secret at `/etc/app/secret/`.

> Verify: inside the Pod, `LOG_LEVEL=debug`, `/etc/app/app.properties` has the two feature lines, `/etc/app/secret/api_key` is `ABCD-1234`, and the Container shows the LimitRange-injected requests/limits (`100m/128Mi`, `500m/512Mi`).

---

## Lab 12 — Scheduling

**⏱ 20 min**

**Objective:** Pin `batch` work to a dedicated worker, spread stateless `web` replicas across workers, and prove a Pod with no toleration cannot reach the dedicated worker.

### Tasks

1. Create namespace `l8-lab`.
2. Label `cka-lab-worker` with `role=web` and `cka-lab-worker2` with `role=batch`.
3. Taint `cka-lab-worker2` with `workload=batch:NoSchedule`.
4. Deploy `batch-job` (1 replica, `busybox:1.37`, command `sleep 600`) with the right toleration + `nodeSelector` so it lands on `cka-lab-worker2`.
5. Deploy `web` (`nginx:1.27`, 2 replicas) with `nodeSelector: role=web` and `podAntiAffinity` so replicas spread — observe one Pod Pending (only one matching node).
6. Re-target `web` so it can run on `role in (web, batch)` and tolerates the batch taint. Both replicas should now schedule on different nodes.
7. Show that without the toleration a Pod targeted to the `batch` node cannot schedule.

> Verify: `batch-job` runs on `cka-lab-worker2`; the two `web` Pods land one per worker; the un-tolerating Pod stays `Pending` with an "untolerated taint {workload: batch}" event.

---

## Lab 13 — Networking & NetworkPolicy

**⏱ 20 min**

**Objective:** Lock down a 3-tier mini-app so only `web` can reach `api`, only `api` can reach `db`, and everything else is denied.

### Tasks

1. Create namespace `l9-lab`.
2. Deploy 3 echo Pods labelled `tier=web`, `tier=api`, `tier=db`, plus matching ClusterIP Services `web`, `api`, `db` on port 80 (add a small busybox tester per tier).
3. Verify everything talks unrestricted (baseline).
4. Add a default-deny ingress policy for the namespace.
5. Add a policy so `api` accepts traffic only from Pods with `tier=web`.
6. Add a policy so `db` accepts traffic only from Pods with `tier=api`.
7. Show that `web → api` works, `web → db` fails, `api → db` works.

> Verify: `web → api` returns "hello from api", `web → db` times out, `api → db` returns "hello from db".

---

## Lab 14 — Security (RBAC)

**⏱ 20 min**

**Objective:** Build a `read-only-operator` ServiceAccount that can read every resource kind in namespace `team-a` plus list nodes cluster-wide — and nothing else. Run a Pod under that identity and prove the limits.

### Tasks

1. Create namespace `team-a`.
2. Create ServiceAccount `read-only-operator` in `team-a`.
3. Grant it the built-in ClusterRole **`view`** scoped to `team-a` (via a RoleBinding).
4. Add a ClusterRole `nodes-viewer` (`nodes`: get/list/watch) and bind it cluster-wide to the SA.
5. Verify with `auth can-i`: list pods in `team-a` → yes; create deployments in `team-a` → no; list nodes cluster-wide → yes; list pods in `default` → no.
6. Launch a `bitnami/kubectl` Pod under the SA and confirm `kubectl get pods` works in `team-a` but is forbidden in `default`.

> Verify: the four `auth can-i` answers are yes / no / yes / no; the Pod can `get pods` in `team-a` and `get nodes`, but is Forbidden in `default`.

---

## Lab 15 — CRDs & Operators

**⏱ 15 min**

**Objective:** Extend the Kubernetes API with a CustomResourceDefinition, create and inspect a custom resource, and articulate what turns a CRD into an operator.

### Tasks

1. Create a **CRD** `widgets.demo.example.com` (group `demo.example.com`, kind `Widget`, plural `widgets`, shortName `wd`, scope `Namespaced`) with a `spec` of `size` (string) and `count` (integer), and `additionalPrinterColumns` showing `Size` and `Count`.
2. Create a `Widget` named `blue-widget` with `size: large`, `count: 5`.
3. Show it with `kubectl get wd` (printer columns visible), read its schema with `kubectl explain widget.spec`, and find the kind via `kubectl api-resources`.
4. In one sentence, state what you would add to make this CRD an **operator**.

> Verify: `kubectl get crd widgets.demo.example.com` is Established; `kubectl get wd` shows `blue-widget   large   5` via the printer columns; the CRD is cluster-scoped while the `Widget` lives in a namespace.

---

## Lab 16 — Node Maintenance & Scaling

**⏱ 35 min**   🖥️

**Objective:** Operate a running Kubespray cluster on Day-2 without disrupting its workloads: add a worker node (append to the inventory under `kube_node`, then `scale.yml --limit=<newnode>`), then remove a node (`remove-node.yml -e node=<node>` — which cordons, drains and resets it — then delete it from the inventory). Note the special procedures for control-plane / etcd nodes and the never-below-3 rule.

### Tasks

**Part A — add a worker (`node6`)**

1. Note the current cluster state (`kubectl get nodes`).
2. Append `node6` to the inventory: add the host and put it **only** under `kube_node` (a worker is *not* in `etcd` or `kube_control_plane`).
3. Refresh the facts cache for **all** nodes first (`facts.yml`, no `--limit`) — so the limited scale run doesn't work from stale facts.
4. `--syntax-check` `scale.yml`.
5. 🖥️ Run `scale.yml` with `--limit=node6` to join only the new worker.
6. Verify `node6` shows up `Ready`.

**Part B — remove a node (`node5`)**

7. 🖥️ Run `remove-node.yml` with `-e node=node5` (cordons + drains + resets it) **while it is still in the inventory**.
8. Delete `node5` from the inventory file.
9. Verify it is gone from `kubectl get nodes`.

**Part C — note the special cases (no run)**

10. Control-plane node → `cluster.yml`, never `scale.yml`; restart `nginx-proxy`.
11. etcd stays **odd**; you can't remove the *first* control-plane/etcd entry without reordering; never run an HA cluster **below 3** control-plane nodes.

> Verify: `node6` joins `Ready` after Part A, `node5` is gone after Part B, and the control plane stays at 3 throughout (only worker capacity changed).

---

## Lab 17 — Upgrades (`upgrade-cluster.yml`)

**⏱ 40 min**   🖥️ *(long run)*

**Objective:** Perform a graceful, rolling upgrade of a Kubespray-managed cluster the safe way: snapshot etcd first, check out the matching Kubespray release tag, bump `kube_version`, then run `upgrade-cluster.yml` so nodes are cordoned/drained/uncordoned one at a time — moving Kubernetes one minor at a time (`1.30 → 1.31 → 1.32`, never skip).

### Tasks

1. 🖥️ Confirm the current cluster state and K8s version.
2. 🖥️ **Take an etcd snapshot first** — your only rollback path.
3. Move Kubespray to the **next** release tag (`git fetch --tags && git checkout`).
4. Reinstall pinned requirements for that tag; diff `group_vars` for drift.
5. Bump **`kube_version`** to the next minor in `group_vars`.
6. `--syntax-check` `upgrade-cluster.yml` before touching nodes.
7. 🖥️ Run the graceful upgrade, **one node at a time** (`-e serial=1`).
8. 🖥️ Verify the node-by-node version progression, then repeat for the next minor.

> Verify: with `serial=1`, nodes flip to the new version one at a time (the cluster stays serviceable); after each minor `kubectl get nodes` is all-`Ready` on the new version and `kubectl version` reports the new server version — repeat to reach 1.32.

---

## Lab 18 — Backup, Recovery & Reset

**⏱ 35 min**   🖥️

**Objective:** Practice the Day-2 "protect and repair" flow: take a verified etcd snapshot with `etcdctl`, recover a broken control plane from that snapshot with `recover-control-plane.yml`, and tear the cluster down cleanly with `reset.yml`. Finish by knowing the toggle for Secret encryption at rest.

### Tasks

1. Identify the etcd client endpoint and the cert trio under `/etc/ssl/etcd/ssl/`.
2. 🖥️ On an etcd node, take a snapshot with `etcdctl snapshot save`.
3. 🖥️ Verify the snapshot with `etcdctl snapshot status --write-out=table`; copy it **off the node**.
4. `--syntax-check` `recover-control-plane.yml` and `reset.yml` before touching anything.
5. Prepare the recovery inventory: `broken_etcd` / `broken_kube_control_plane` groups, survivors listed first.
6. 🖥️ Run `recover-control-plane.yml` scoped to broken + healthy nodes, with `-e etcd_retries=...` and the snapshot path.
7. 🖥️ Tear down with `reset.yml -e reset_confirmation=yes`.
8. Note the encryption-at-rest toggle for later.

> Verify: `snapshot save` prints a success line and `snapshot status --write-out=table` shows a non-zero `HASH | REVISION | TOTAL KEYS | TOTAL SIZE` row; after recovery the broken etcd member rejoins (member list / endpoint health healthy, nodes/pods healthy); after `reset.yml` the target nodes have no kubelet/etcd/CNI left.

---

## Lab 19 — Add-ons & Packaging

**⏱ 25 min**

**Objective:** Two exercises: **(A)** turn on a set of Kubespray add-ons declaratively in one inventory file (`metrics_server`, `helm`, `cert_manager`, `metallb`, `local_volume_provisioner`, `registry`), re-run the playbook, and verify each; **(B)** install the same app two ways — once with **Helm** (override a value, roll back), once with **Kustomize** (a prod overlay over a base).

### Tasks

**Part A — Kubespray add-ons**

1. Enable the simple toggles in `addons.yml`: `metrics_server_enabled`, `helm_enabled`, `cert_manager_enabled`, `local_volume_provisioner_enabled`, `registry_enabled`.
2. Enable **MetalLB** and define an **address pool** in `metallb_config` (e.g. `192.168.121.240-192.168.121.250` — change to match your subnet); set the strict-ARP prerequisite.
3. Confirm `addons.yml` is **valid YAML** before running anything.
4. 🖥️ Apply the add-ons — re-run `cluster.yml`, or just `--tags=apps`.
5. 🖥️ Verify: Pods `Running`, `kubectl top nodes`, the `IPAddressPool`, and a `type: LoadBalancer` Service getting an IP.
6. Note the **removed ingress add-ons** → install with Helm post-deploy.

**Part B — Helm + Kustomize**

7. **Helm:** scaffold a chart `site` (`helm create`), install it as release `site` in namespace `helm-lab` with **3 replicas** via `--set`. Upgrade it to image tag `1.27`, then **roll back** to revision 1, and verify the replica count and revision history.
8. **Kustomize:** create a `base` (an nginx Deployment + Service) and a `staging` overlay that sets namespace `kz-staging`, a `stg-` name prefix, `replicas: 2`, and image tag `1.27-alpine`; render with `kubectl kustomize`, then apply with `kubectl apply -k`.

> Verify: all add-on Pods `Running`, `kubectl top nodes` serves metrics, the `IPAddressPool` lists the configured range, and a `type: LoadBalancer` Service gets an `EXTERNAL-IP` from the pool; Helm release `site` is deployed at 3 replicas after the rollback; the Kustomize overlay produces `stg-web` in `kz-staging` with 2 replicas and image `nginx:1.27-alpine`.

---

## Lab 20 — Troubleshooting

**⏱ 20 min**

**Objective:** Triage a small "outage" — a namespace ships three broken Deployments. Find each root cause from `kubectl` output alone, then fix each with the smallest possible change.

> Setup — apply this first, then triage:
> ```bash
> kubectl create ns l11-lab
>
> # A — missing image
> kubectl -n l11-lab create deployment broken-image --image=nginx:9.99-nope
>
> # B — crashloop (the command exits)
> cat <<'EOF' | kubectl apply -f -
> apiVersion: apps/v1
> kind: Deployment
> metadata: {name: crashloop, namespace: l11-lab}
> spec:
>   replicas: 1
>   selector: {matchLabels: {app: crashloop}}
>   template:
>     metadata: {labels: {app: crashloop}}
>     spec:
>       containers:
>       - name: app
>         image: busybox:1.37
>         command: ["sh","-c","echo starting; exit 1"]
> EOF
>
> # C — Pending due to impossible nodeSelector
> cat <<'EOF' | kubectl apply -f -
> apiVersion: apps/v1
> kind: Deployment
> metadata: {name: unschedulable, namespace: l11-lab}
> spec:
>   replicas: 1
>   selector: {matchLabels: {app: unsched}}
>   template:
>     metadata: {labels: {app: unsched}}
>     spec:
>       nodeSelector: {disktype: ssd}     # no node has this label
>       containers: [{name: app, image: nginx:1.27}]
> EOF
> ```

### Tasks

1. List all Pods in `l11-lab` and note their statuses.
2. For each one, identify the root cause using **only** `kubectl`. Write down the command you used.
3. Fix each Deployment with the smallest possible change so all Pods become `Running`.

> Verify: all Pods `1/1 Running` with a stable restart count.

---

## Lab 21 — Hardening & Offline

**⏱ 20 min**

**Objective:** Turn a plain Kubespray config into a hardened, kube-bench-aligned one by applying the upstream CI hardening scenario `tests/files/ubuntu24-calico-all-in-one-hardening.yml` as your `group_vars` base (admission plugins + Pod Security Admission, audit logging, encryption at rest, anonymous access off, kubelet lock-down). Validate and `--syntax-check` it, deploy it (🖥️), and prove the hardening took effect. The lab also covers large-deployment tuning and the offline/air-gapped vars.

### Tasks

1. Drop the CI **hardening** config in as your `k8s_cluster` group_vars and read what it turns on (admission/PSA, audit, encryption, kubelet, anon off).
2. Confirm the config is **valid YAML** and the key hardening vars parse to the values you expect.
3. `--syntax-check` `cluster.yml` with the hardened vars in place.
4. Add the **large-deployment** tuning (`download_run_once`, ansible `forks`, a dedicated events etcd) — config-only here, no nodes.
5. Sketch the **offline/air-gapped** vars from `group_vars/all/offline.yml` and where `contrib/offline/` fits.
6. 🖥️ Deploy the hardened cluster with `cluster.yml` (use `--forks` to show the scale flag).
7. 🖥️ **Verify hardening:** apiserver flags, PodSecurity namespace labels, audit log present, read-only port closed.

> Verify: apiserver shows `--anonymous-auth=false` and `--profiling=false`, `PodSecurity` + `EventRateLimit` in `--enable-admission-plugins`, a privileged Pod is Forbidden by the `restricted` PSA default, the audit log grows, the kubelet read-only port (10255) does not listen, and controller-manager (10257) / scheduler (10259) bind only to `127.0.0.1`.

---

## Lab 22 — Cloud & OpenStack

**⏱ 25 min**   🖥️

**Objective:** Take a working Kubespray inventory and wire the cluster into an OpenStack tenant the modern, supported way: the external cloud-controller-manager (`cloud_provider: external` + `external_cloud_provider: openstack`), Keystone application credentials, Octavia for `LoadBalancer` Services, and the Cinder CSI driver for dynamic block storage. Then re-run `cluster.yml` and verify the integration.

### Tasks

1. (Optional) Provision the VM fleet on OpenStack with `contrib/terraform/openstack`.
2. Start from a fresh inventory copy and gather the OpenStack IDs you'll need.
3. Enable the **external cloud provider** in `group_vars/all/all.yml`.
4. Fill **auth + Octavia LBaaS** in `group_vars/all/openstack.yml` (application credentials, auth URL/region, floating/subnet network IDs, amphora provider).
5. Enable **Cinder CSI** and a default `storage_classes` entry.
6. `--syntax-check` `cluster.yml` with the new vars in place.
7. **Source your openstack-rc**, then (re-)run `cluster.yml` to roll out the CCM and Cinder CSI.
8. 🖥️ Verify the **cloud-controller** and **cinder-csi** Pods are `Running`.
9. 🖥️ Verify a `type: LoadBalancer` Service gets an **Octavia floating IP**.
10. 🖥️ Verify a **PVC** binds and provisions a **Cinder volume**.

> Verify: `openstack-cloud-controller-manager` and the `csi-cinder-*` Pods are `Running`; `kubectl get storageclass` shows `cinder-csi` as default (`cinder.csi.openstack.org`); the `LoadBalancer` Service's `EXTERNAL-IP` is populated from the floating network; the PVC reaches `Bound` with the volume mounted in the Pod.

---

## Lab 23 — Practice Exam

**⏱ 60 min**

**Objective:** Two independent, exam-style sequences covering the full set of domains end-to-end. Start each task by switching context with `kubectl config set-context --current --namespace=<task-ns>`.

### Practice Exam #1

**Task 1.1 — Build & expose** (namespace `pe1-app`): create Deployment `web` (`nginx:1.27`, 3 replicas, label `tier=frontend`); expose it as ClusterIP Service `web` on port 80; add a NetworkPolicy allowing ingress to `web` **only** from Pods labelled `role=tester` in the same namespace.

**Task 1.2 — Storage** (namespace `pe1-storage`): create a 200 Mi PVC `data` on the default StorageClass; run Pod `writer` (`busybox:1.37`) mounting it at `/data` that writes `done` into `/data/marker` and sleeps; after it is Ready, delete it and run Pod `reader` that reads `/data/marker` to stdout — confirm the value persisted.

**Task 1.3 — Scheduling** (namespace `pe1-sched`): label `cka-lab-worker` with `tier=gold`; taint `cka-lab-worker2` with `dedicated=batch:NoSchedule`; create Pod `payments` (`nginx:1.27`) that must run on a `tier=gold` node, tolerates the taint, and has requests `cpu=100m/memory=128Mi`, limits `cpu=500m/memory=512Mi`.

**Task 1.4 — Troubleshooting** (namespace `pe1-broken`): a Deployment `broken` won't roll out. Diagnose, then make it `1/1 Available` with minimum YAML changes.

> Setup for Task 1.4 — apply first:
> ```bash
> kubectl create ns pe1-broken
> cat <<'EOF' | kubectl apply -f -
> apiVersion: apps/v1
> kind: Deployment
> metadata: {name: broken, namespace: pe1-broken}
> spec:
>   replicas: 1
>   selector: {matchLabels: {app: broken}}
>   template:
>     metadata: {labels: {app: broken}}
>     spec:
>       nodeSelector: {disktype: nvme}     # bug: no node labelled nvme
>       containers:
>       - name: app
>         image: nginx:1.27
>         resources:
>           requests: {cpu: "8",    memory: "16Gi"}    # bug: too big
>           limits:   {cpu: "8",    memory: "16Gi"}
> EOF
> ```

### Practice Exam #2

**Task 2.1 — RBAC & ServiceAccount** (namespaces `pe2-rbac`, `pe2-app`): create ServiceAccount `auditor`; give it list/watch (read-only) on every resource in `pe2-rbac` and `pe2-app` only; run Pod `auditor-pod` as that SA (`bitnami/kubectl:latest`) and verify it can list pods in both namespaces but not in `default`.

**Task 2.2 — etcd snapshot:** take a snapshot named `/tmp/exam-snap.db` inside the etcd Pod and verify it with `snapshot status`.

**Task 2.3 — Node drain:** cordon and drain `cka-lab-worker2` (ignore DaemonSets, delete emptyDir data); confirm all user Pods moved off it; uncordon afterwards.

**Task 2.4 — Ingress** (namespace `pe2-ing`): deploy `echo` (`hashicorp/http-echo:1.0`, 2 replicas, args `-text=v2`, listen `:5678`); expose it as Service `echo` (port 80 → 5678); create an Ingress routing `Host: echo.local` to it via the `nginx` class; verify with `curl -H 'Host: echo.local'` to the controller Pod IP.

> Verify: exam #1 — `web` exposed and policy in place, `reader` prints `done`, `payments` on the gold node, `broken` Running. Exam #2 — `auditor-pod` lists pods in `pe2-app` but is Forbidden in `default`, the snapshot status row prints, `cka-lab-worker2` returns Ready, and the ingress curl returns `v2`.
