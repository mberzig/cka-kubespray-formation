# Lesson 21 — Hardening & Offline — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Objective

Turn a plain Kubespray config into a **hardened, kube-bench-aligned** one by
applying the upstream CI hardening scenario
`tests/files/ubuntu24-calico-all-in-one-hardening.yml` as your `group_vars`
base: admission plugins + Pod Security Admission, audit logging, encryption at
rest, anonymous access off, and the kubelet lock-down knobs. You then validate
the config and `--syntax-check` it (✅), deploy it (🖥️), and **prove** the
hardening took effect by inspecting the apiserver flags, the PodSecurity
namespace labels, and the audit log. The lab also shows the **large-deployment**
tuning (`download_run_once`, ansible `forks`, a dedicated **events** etcd) and
the **offline/air-gapped** vars (`group_vars/all/offline.yml`,
`contrib/offline/`) you bolt on for scale and air-gapped sites.

## Prerequisites

- Control node set up (`setup.md`): `ksenv` venv + `pip install -r requirements.txt`
  (ansible-core 2.18.x), Kubespray checked out, `inventory/mycluster` created
  from `inventory/sample`.
- Hardening requires **Kubernetes ≥ v1.23.6** (default in current Kubespray) and
  that **no other group_vars override** the hardening keys.
- (For the 🖥️ steps only) a reachable fleet in
  `inventory/mycluster/inventory.ini`. The hardening scenario is `mode: all-in-one`,
  so a **single node** in all three groups (`kube_control_plane`, `etcd`,
  `kube_node`) is enough for the lab.
- 📖 Reference docs:
  [Hardening](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/hardening.md) ·
  [Large deployments](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/large-deployments.md) ·
  [Offline environment](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/offline-environment.md) ·
  [CI hardening config](https://github.com/kubernetes-sigs/kubespray/blob/master/tests/files/ubuntu24-calico-all-in-one-hardening.yml)

## Steps (énoncé)

1. (✅) Drop the CI **hardening** config in as your `k8s_cluster` group_vars and
   read what it turns on (admission/PSA, audit, encryption, kubelet, anon off).
2. (✅) Confirm the config is **valid YAML** and the key hardening vars parse to
   the values you expect.
3. (✅) `--syntax-check` `cluster.yml` with the hardened vars in place.
4. (📖) Add the **large-deployment** tuning (run-once download, forks, events
   etcd) — config-only here, no nodes.
5. (📖) Sketch the **offline/air-gapped** vars from `group_vars/all/offline.yml`
   and where `contrib/offline/` fits.
6. (🖥️) Deploy the hardened cluster with `cluster.yml` (use `--forks` to show
   the scale flag).
7. (🖥️) **Verify hardening:** apiserver flags, PodSecurity namespace labels /
   `psa`, audit log present, read-only port closed.

## Solution

```bash
export LC_ALL=C.UTF-8 LANG=C.UTF-8            # avoid the locale error
source ksenv/bin/activate && cd kubespray

# 1 — apply the CI hardening scenario as the k8s_cluster group_vars (✅)
#     This is a complete, CI-validated hardened config — copy it in as your base.
cp tests/files/ubuntu24-calico-all-in-one-hardening.yml \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
#     Skim what it enables (anon off, audit, encryption, PSA, kubelet lockdown):
grep -E 'remove_anonymous_access|kubernetes_audit|kube_encrypt_secret_data|kube_pod_security|PodSecurity|kube_read_only_port|kube_profiling' \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml

# 2 — YAML validity + spot-check the values that matter (✅ validated)
python3 -c "import yaml,sys; d=yaml.safe_load(open('inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml')); \
print('keys:', len(d)); \
assert d['remove_anonymous_access'] is True; \
assert d['kubernetes_audit'] is True; \
assert d['kube_encrypt_secret_data'] is True; \
assert d['kube_read_only_port'] == 0; \
assert d['kube_profiling'] is False; \
assert d['kube_pod_security_default_enforce']=='restricted'; \
assert 'PodSecurity' in d['kube_apiserver_enable_admission_plugins']; \
print('hardening keys OK')"

# 3 — syntax-check cluster.yml with the hardened vars in place (✅ validated: passes)
ansible-playbook -i inventory/mycluster/inventory.ini --syntax-check playbooks/cluster.yml

# ---- 4 & 5 are 📖 config-only (no nodes); 6 & 7 are 🖥️ (need the fleet) ----

# 6 — deploy the HARDENED cluster (🖥️ needs the fleet; ~15-25 min)
#     --forks is the large-deployment parallelism knob; --timeout helps slow nodes.
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu -b \
  --private-key ~/.ssh/id_rsa --forks 10 --timeout 600 playbooks/cluster.yml

# kubeconfig (set kubeconfig_localhost: true before deploy, or copy admin.conf):
scp ubuntu@<cp1-ip>:/etc/kubernetes/admin.conf ~/.kube/config

# 7 — VERIFY HARDENING (🖥️ on the deployed node) — see the Verification section.
```

### Step 1 — what the hardening config actually turns on (📖)

The CI file `tests/files/ubuntu24-calico-all-in-one-hardening.yml` is the
reference hardened `group_vars`. The load-bearing blocks (verbatim from upstream):

```yaml
# kube-apiserver: authz, anon off, request timeout, audit, TLS, encryption
authorization_modes: ["Node", "RBAC"]
remove_anonymous_access: true                 # turns anonymous-auth off (RBAC is on)
kube_apiserver_service_account_lookup: true
kube_apiserver_request_timeout: 120s
kubernetes_audit: true
audit_log_path: "/var/log/kube-apiserver-log.json"
audit_log_maxage: 30
audit_log_maxbackups: 10
audit_log_maxsize: 100
tls_min_version: VersionTLS12
tls_cipher_suites:
  - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
kube_encrypt_secret_data: true                # encryption at rest for secrets in etcd
kube_encryption_resources: [secrets]
kube_encryption_algorithm: "secretbox"
kube_profiling: false                         # /debug/pprof off

# admission plugins + EventRateLimit config + PodSecurity
kube_apiserver_enable_admission_plugins:
  - EventRateLimit
  - AlwaysPullImages
  - ServiceAccount
  - NamespaceLifecycle
  - NodeRestriction
  - LimitRanger
  - ResourceQuota
  - MutatingAdmissionWebhook
  - ValidatingAdmissionWebhook
  - PodNodeSelector
  - PodSecurity
kube_apiserver_admission_control_config_file: true   # needed by EventRateLimit/PodSecurity
kube_apiserver_admission_event_rate_limits:
  limit_1: { type: Namespace, qps: 50, burst: 100, cache_size: 2000 }
  limit_2: { type: User, qps: 50, burst: 100 }

# controller-manager / scheduler bind to loopback only
kube_controller_manager_bind_address: 127.0.0.1
kube_controller_terminated_pod_gc_threshold: 50
kube_controller_feature_gates: ["RotateKubeletServerCertificate=true"]
kube_scheduler_bind_address: 127.0.0.1

# kubelet lock-down
kubelet_authentication_token_webhook: true
kube_read_only_port: 0                         # close the unauthenticated read-only port
kubelet_rotate_certificates: true
kubelet_rotate_server_certificates: true
kubelet_protect_kernel_defaults: true
kubelet_event_record_qps: 1
kubelet_streaming_connection_idle_timeout: "5m"
kubelet_make_iptables_util_chains: true
kubelet_feature_gates: ["RotateKubeletServerCertificate=true"]
kubelet_seccomp_default: true
kubelet_systemd_hardening: true
kube_owner: root
kube_cert_group: root

# Pod Security Admission: restricted enforce by default (kube-system exempt)
kube_pod_security_use_default: true
kube_pod_security_default_enforce: restricted
```

> The real CI file also sets `etcd_deployment_type: kubeadm` and a
> `kubelet_csr_approver_*` block (it's a test/conformance config). The
> hardening **docs** suggest `etcd_deployment_type: host` to isolate etcd; pick
> per your topology. `kubelet_authorization_mode_webhook` (Webhook authz on the
> kubelet) is also a docs hardening knob — add it if you want full kube-bench
> coverage. These values map directly to **CIS Kubernetes Benchmark** checks, so
> run **kube-bench** after install to confirm.

### Step 4 — large-deployment tuning (📖 config-only)

For hundreds of nodes you scale **Ansible** and the **cluster**. The Ansible-side
flags are command-line (`--forks`, `--timeout`) plus a couple of group_vars; the
cluster-side ones go in `group_vars`:

```bash
# Ansible side — on the cluster.yml command line:
ansible-playbook ... --forks 50 --timeout 600 playbooks/cluster.yml
```

```yaml
# group_vars/all/all.yml — download once, push out (don't make every node hit the net)
download_run_once: true
download_localhost: true        # cache on the control node
retry_stagger: 60               # spread retries so they don't all hit the CP at once

# group_vars/k8s_cluster/k8s-cluster.yml — cluster-side scale knobs
etcd_events_cluster_setup: true # store events in a SEPARATE, dedicated etcd cluster
# longer node-monitor / eviction timeouts so transient pressure doesn't evict good nodes:
kube_controller_node_monitor_grace_period: 60s
kube_controller_node_monitor_period: 10s
kube_apiserver_pod_eviction_not_ready_timeout_seconds: 60
kube_apiserver_pod_eviction_unreachable_timeout_seconds: 60
# CoreDNS scaling for a big fleet:
dns_replicas: 3
```

### Step 5 — offline / air-gapped vars (📖 config-only)

In an air-gapped site you mirror everything Kubespray would pull and point
`group_vars/all/offline.yml` at the mirrors. Generate the asset list with
`contrib/offline/`:

```bash
# 📖 list every image/file Kubespray needs, to mirror them (run from repo root)
bash contrib/offline/generate_list.sh        # writes temp/{files,images}.list
# see contrib/offline/README.md for the registry + HTTP server setup
```

```yaml
# inventory/mycluster/group_vars/all/offline.yml — point at your mirrors
registry_host: "myinternalregistry:5000"
files_repo: "http://myinternalfileserver:8080"
ubuntu_repo: "http://myinternalmirror/ubuntu"

# images → internal registry
kube_image_repo: "{{ registry_host }}"
gcr_image_repo: "{{ registry_host }}"
docker_image_repo: "{{ registry_host }}"
quay_image_repo: "{{ registry_host }}"
github_image_repo: "{{ registry_host }}"

# binaries → internal HTTP file server
github_url: "{{ files_repo }}/github.com"
dl_k8s_io_url: "{{ files_repo }}/dl.k8s.io"
storage_googleapis_url: "{{ files_repo }}/storage.googleapis.com"
get_helm_url: "{{ files_repo }}/get.helm.sh"

# containerd registry mirror so the runtime pulls from the internal registry
containerd_registries_mirrors:
  - prefix: "{{ registry_host }}"
    mirrors:
      - host: "{{ registry_host }}"
        capabilities: ["pull", "resolve"]
        skip_verify: true
```

> `download_run_once: true` (Step 4) pairs naturally with offline: download/cache
> once, then push to every node. Put `registry_pass`/`files_repo_pass` behind
> **Ansible Vault**.

## Verification

**✅ Validated on the control node (no fleet needed):**

```text
$ python3 -c "import yaml; ... print('hardening keys OK')"
keys: 40+
hardening keys OK                              # YAML valid; key values as expected

$ grep -E 'remove_anonymous_access|kubernetes_audit|...' .../k8s-cluster.yml
remove_anonymous_access: true
kubernetes_audit: true
kube_encrypt_secret_data: true
kube_read_only_port: 0
kube_profiling: false
kube_pod_security_default_enforce: restricted   # PSA is in the admission plugin list

$ ansible-playbook ... --syntax-check playbooks/cluster.yml
playbook: playbooks/cluster.yml                # exit 0 — syntax OK with hardened vars
```

**🖥️ After the hardened deploy (described, not validated)** — prove each control
took effect:

```text
# (a) apiserver flags: anonymous-auth off, profiling off, PodSecurity + EventRateLimit on,
#     audit + encryption configured. Inspect the static-pod manifest on a control-plane node:
$ sudo grep -E 'anonymous-auth|profiling|enable-admission-plugins|audit-log-path|encryption-provider-config' \
    /etc/kubernetes/manifests/kube-apiserver.yaml
    --anonymous-auth=false
    --profiling=false
    --enable-admission-plugins=...,PodSecurity,EventRateLimit,...
    --audit-log-path=/var/log/kube-apiserver-log.json
    --encryption-provider-config=/etc/kubernetes/ssl/secrets_encryption.yaml

# (b) Pod Security Admission default = restricted. Default cluster config + per-ns labels:
$ kubectl label ns default --list | grep pod-security    # OR check the admission config
pod-security.kubernetes.io/enforce=restricted            # kube-system is exempt by design
#   Functional proof — a privileged Pod is REJECTED by PSA:
$ kubectl run nginx --image=nginx --privileged
Error from server (Forbidden): ... violates PodSecurity "restricted:latest": privileged

# (c) audit log is being written on the control-plane node:
$ sudo ls -l /var/log/kube-apiserver-log.json
-rw------- 1 root root 1.2M ... /var/log/kube-apiserver-log.json
$ sudo tail -1 /var/log/kube-apiserver-log.json | head -c 120
{"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"Metadata",...

# (d) encryption at rest — Secrets are stored encrypted (secretbox), not plaintext, in etcd
$ kubectl -n kube-system get secret ... -o jsonpath='{.data}' >/dev/null && echo "API serves it decrypted"
#   raw etcd value is prefixed k8s:enc:secretbox:... (not 'k8s:...' plaintext)

# (e) kubelet read-only port closed (10255 must NOT listen), authn webhook on:
$ sudo ss -tlnp | grep ':10255' || echo "read-only port closed (good)"
read-only port closed (good)

# (f) controller-manager / scheduler bound to loopback only:
$ sudo ss -tlnp | grep -E ':10257|:10259'
127.0.0.1:10257   ...   # controller-manager — loopback, not 0.0.0.0
127.0.0.1:10259   ...   # scheduler

# (g) optional but recommended — run kube-bench for the full CIS report:
$ kubectl run kube-bench --image=aquasec/kube-bench:latest --rm -it --restart=Never -- --version 1.x
... [PASS] checks for anonymous-auth, read-only-port, audit, encryption, profiling ...
```

**Pass criteria:** `--anonymous-auth=false` and `--profiling=false` on the
apiserver; `PodSecurity` + `EventRateLimit` in `--enable-admission-plugins`; a
privileged Pod is **Forbidden** by the `restricted` PSA default; the audit log
file exists and grows; the kubelet read-only port (10255) does **not** listen;
controller-manager (10257) and scheduler (10259) bind only to `127.0.0.1`.

## Faster lab variant — single node

The hardening scenario is `mode: all-in-one`, so you don't need a fleet for the
control logic — one Ubuntu 24.04 VM in all three groups (`kube_control_plane`,
`etcd`, `kube_node`) runs the whole hardened deploy and every verification above.

## Cleanup

```bash
# Restore a plain config (drop hardening) and/or reset the cluster (🖥️):
ansible-playbook -i inventory/mycluster/inventory.ini -b \
  -e reset_confirmation=yes playbooks/reset.yml

# Control node: revert group_vars back to the sample if you want a clean base (✅):
cp inventory/sample/group_vars/k8s_cluster/k8s-cluster.yml \
   inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```
