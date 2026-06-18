# Lesson 15 — CRDs & Operators — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts

- A **custom resource** extends the Kubernetes API with a new object kind you can
  manage with `kubectl` like a built-in. Two ways to add one:
  - **CustomResourceDefinition (CRD)** — declarative, **no code**. The common path.
  - **API Aggregation** — run your own apiserver. More flexible, needs code.
- A **CRD** object (`apiextensions.k8s.io/v1`) registers the new kind; its
  `metadata.name` **must** be `<plural>.<group>`. A CRD is **cluster-scoped**,
  even when the resources it defines are `Namespaced`.
- A **controller** runs a **control loop**: observe `.spec` → diff vs actual →
  act → update `.status`.
- An **Operator** = **CRD(s) + a custom controller** that encodes operational
  knowledge (deploy, back up, upgrade an app). You install it (manifests/Helm),
  then drive it with custom resources.

---

## Examples — CustomResourceDefinitions

### 1. Define a CRD

`crontab-crd.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com    # MUST be <plural>.<group>
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true       # exposed via the API
      storage: true      # exactly one version is the storage version
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec: {type: string}
                image:    {type: string}
                replicas: {type: integer}
      additionalPrinterColumns:
        - {name: Spec,  type: string, jsonPath: .spec.cronSpec}
        - {name: Image, type: string, jsonPath: .spec.image}
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames: ["ct"]
```

```bash
kubectl apply -f crontab-crd.yaml
kubectl get crd crontabs.stable.example.com
```

Sample output:

```
customresourcedefinition.apiextensions.k8s.io/crontabs.stable.example.com created
NAME                          CREATED AT
crontabs.stable.example.com   2026-06-15T22:17:01Z
```

### 2. Create a custom resource of the new kind

`my-crontab.yaml`:

```yaml
apiVersion: stable.example.com/v1     # <group>/<version>
kind: CronTab                          # matches spec.names.kind
metadata: {name: my-new-cron-object}
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  replicas: 3
```

```bash
kubectl apply -f my-crontab.yaml
kubectl get ct                          # short name; printer columns show
kubectl explain crontab.spec
```

Sample output:

```
crontab.stable.example.com/my-new-cron-object created
NAME                 SPEC          IMAGE
my-new-cron-object   * * * * */5   my-awesome-cron-image
GROUP:      stable.example.com
KIND:       CronTab
FIELD: spec <Object>
FIELDS:
  cronSpec  <string>
  image     <string>
  replicas  <integer>
```

The `additionalPrinterColumns` surface `Spec`/`Image` in `kubectl get`, the
`shortNames` give `ct`, and `kubectl explain` reads the schema. The structural
`openAPIV3Schema` (mandatory in `v1`) validates each custom resource.

### 3. Discover CRDs and their kinds

```bash
kubectl get crd                          # all CRDs (cluster-scoped)
kubectl api-resources | grep stable      # find kinds from a CRD's group
kubectl get crontabs -A                  # custom objects, all namespaces
```

---

## Examples — Operators (reference walkthrough)

### 4. Install an operator (e.g. cert-manager) and inspect it

Installing an operator typically adds **CRDs + a controller Deployment + RBAC**.
With Helm (Lesson 12):

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace --set crds.enabled=true
```

Then inspect what the operator added:

```bash
kubectl get crd | grep cert-manager.io        # the new custom kinds
kubectl -n cert-manager get deploy             # the controller(s)
kubectl api-resources --api-group=cert-manager.io
```

### 5. Drive the operator with custom resources

You no longer create low-level objects by hand — you declare the **custom
resource** and the controller reconciles it:

```bash
kubectl get clusterissuers,certificates -A     # cert-manager's custom kinds
kubectl describe certificate <name> -n <ns>    # controller updates .status
```

> The same pattern applies to the Prometheus Operator, database operators, etc.:
> install (CRDs + controller + RBAC), then create custom resources; the
> control loop does the operational work.

---

## Lab

**Goal**: define a CRD, create and inspect a custom resource, and reason about the
operator pattern.

### Steps

1. Create a **CRD** `widgets.demo.example.com` (group `demo.example.com`, kind
   `Widget`, plural `widgets`, shortName `wd`, scope `Namespaced`) with a `spec`
   of `size` (string) and `count` (integer), and an `additionalPrinterColumns`
   showing `Size` and `Count`.
2. Create a `Widget` named `blue-widget` with `size: large`, `count: 5`.
3. Show it with `kubectl get wd` (printer columns), `kubectl explain widget.spec`,
   and find the kind via `kubectl api-resources`.
4. State, in one line, what you would add to turn this CRD into an **operator**.

### Solution

```bash
# 1
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

# Wait for the API server to register the new type before creating a CR
# (without this the next apply can race with "no matches for kind Widget").
kubectl wait --for condition=established crd/widgets.demo.example.com --timeout=60s

# 2
cat <<'EOF' | kubectl apply -f -
apiVersion: demo.example.com/v1
kind: Widget
metadata: {name: blue-widget}
spec: {size: large, count: 5}
EOF

# 3
kubectl get wd
kubectl explain widget.spec
kubectl api-resources | grep demo.example.com
```

4. **Answer:** add a **custom controller** (a Deployment running a reconcile loop
   that watches `Widget` objects and acts on them) → CRD + controller = an
   **operator**.

### Verification

```bash
kubectl get crd widgets.demo.example.com           # Established
kubectl get wd blue-widget                          # SIZE large, COUNT 5
```

Expected: `kubectl get wd` shows `blue-widget   large   5` via the printer
columns; the CRD is cluster-scoped while the `Widget` lives in a namespace.

### Cleanup

```bash
kubectl delete widget blue-widget
kubectl delete crd widgets.demo.example.com
# If you installed cert-manager for §4: helm uninstall cert-manager -n cert-manager
```
