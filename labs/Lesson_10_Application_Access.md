# Lesson 10 — Application Access — Lab

> 🖥️ **Runs on the shared lab cluster** — a multi-node Kubernetes cluster deployed on
> **OpenStack via Kubespray** (see `setup.md`). Infra steps run from the **Ansible
> control node** against `inventory/lab/`; `kubectl` steps use the cluster's
> `admin.conf`. Substitute your real node names (`kubectl get nodes`) for any
> `cka-lab-*` / `node1` examples. *(Usage-lab mechanics validated on kind; Kubespray
> steps validated via docs + `--syntax-check`.)*

## Concepts

| Service type     | What it exposes                                                          |
|------------------|--------------------------------------------------------------------------|
| `ClusterIP`      | Cluster-internal virtual IP. Default. Reachable from any Pod by DNS.     |
| `NodePort`       | All nodes open the same TCP port (30000-32767). Reachable from outside.  |
| `LoadBalancer`   | NodePort + cloud LB. On bare-metal needs MetalLB or similar.             |
| Headless (`clusterIP: None`) | Returns Pod IPs directly via DNS — used for StatefulSets.   |
| `ExternalName`   | DNS CNAME to an external host.                                            |

**Ingress** is HTTP(S)-aware routing (host/path → backend Service). It requires
an **ingress controller** (nginx, traefik, HAProxy, cloud-native) to actually
serve traffic.

DNS pattern inside a cluster:

```
<service>.<namespace>.svc.cluster.local
```

A Pod in namespace `X` can reach `web` in namespace `Y` with `web.Y` (search
list does the rest).

---

## Setup (one-time, for Ingress examples)

On the **OpenStack lab cluster** use the **cloud** provider manifest — its
controller Service is `type: LoadBalancer`, so **Octavia** gives the ingress a
real **floating IP** (no node label / hostPort hack like kind needs):

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml
kubectl rollout status -n ingress-nginx deployment/ingress-nginx-controller
kubectl -n ingress-nginx get svc ingress-nginx-controller   # EXTERNAL-IP = Octavia floating IP
```

> On `kind` instead use `…/provider/kind/deploy.yaml` and
> `kubectl label node <control-plane> ingress-ready=true` (kind has no cloud LB).
> Below you can reach the Ingress either via the controller **Pod IP** (shown) or,
> on this cluster, via the controller's **EXTERNAL-IP** (`curl -H 'Host: …' http://<EXTERNAL-IP>/`).

## Examples

### 1. Create a deployment and expose it three ways

```bash
kubectl create ns l5
kubectl -n l5 create deployment web --image=nginx:1.27 --replicas=2
kubectl -n l5 wait --for=condition=Available deployment/web

# ClusterIP (default)
kubectl -n l5 expose deployment web --port=80 --name=web-clusterip
# NodePort
kubectl -n l5 expose deployment web --port=80 --name=web-nodeport --type=NodePort

kubectl -n l5 get svc
```

Sample output:

```
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
web-clusterip   ClusterIP   10.96.101.87   <none>        80/TCP
web-nodeport    NodePort    10.96.83.131   <none>        80:31539/TCP
```

### 2. Verify in-cluster connectivity

```bash
kubectl -n l5 run client --image=busybox:1.37 --rm -it --restart=Never -- \
  wget -qO- http://web-clusterip/ | head -3
```

Expected: `<!DOCTYPE html>` + nginx welcome.

### 3. DNS lookup

```bash
kubectl -n l5 run dnstest --image=busybox:1.37 --rm -it --restart=Never -- \
  nslookup web-clusterip.l5.svc.cluster.local
```

The FQDN resolves to the Service's ClusterIP. Use the FQDN to avoid relying on
`resolv.conf` search ordering.

### 4. Endpoints — see actual backends

```bash
kubectl -n l5 get endpoints web-clusterip
# Should list one <PodIP>:80 per replica.
```

### 5. Headless Service (no virtual IP — Pod IPs directly)

```yaml
apiVersion: v1
kind: Service
metadata: {name: headless, namespace: l5}
spec:
  clusterIP: None
  selector: {app: web}
  ports: [{port: 80}]
```

A DNS lookup for `headless.l5.svc.cluster.local` returns **A records for each
Pod IP** instead of a single ClusterIP. Required for StatefulSets.

### 6. Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: web, namespace: l5}
spec:
  ingressClassName: nginx
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-clusterip
            port: {number: 80}
```

Test it (without DNS entry, use `curl --resolve`):

```bash
INGRESS_IP=$(kubectl -n ingress-nginx get pod \
  -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].status.podIP}')

kubectl run curl-test --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  sh -c "curl -s -H 'Host: web.local' http://$INGRESS_IP/ | head -5"
```

Returns the nginx welcome page → request was routed by host header through the
ingress controller to the `web-clusterip` Service which load-balanced to a Pod.

### 7. `kubectl port-forward` (debug / local access)

```bash
kubectl -n l5 port-forward svc/web-clusterip 8080:80
# Now `curl http://localhost:8080/` from your host hits the Service.
```

---

## Lab

**Goal**: expose a single Deployment three ways and observe each.

### Steps

1. Create namespace `l5-lab`. Deploy `hello` from `hashicorp/http-echo:1.0`
   listening on `:5678` with `-text=hello-from-pod-$(POD_NAME)` (use the
   downward API). 2 replicas.
2. Create a ClusterIP Service `hello` exposing port 80 → 5678.
3. Create a NodePort Service `hello-np` for the same Deployment.
4. From a Pod inside the cluster, `curl http://hello/` 5 times — observe round
   robin between Pods.
5. Create an Ingress `hello-ing` routing `Host: hello.local` to `hello:80`.

### Solution

```bash
# 1
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

# 2
kubectl -n l5-lab expose deployment hello --name=hello --port=80 --target-port=5678

# 3
kubectl -n l5-lab expose deployment hello --name=hello-np --port=80 --target-port=5678 --type=NodePort

# 4
kubectl -n l5-lab run loop --image=busybox:1.37 --rm -it --restart=Never -- \
  sh -c 'for i in 1 2 3 4 5; do wget -qO- http://hello/; done'

# 5
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
        backend:
          service:
            name: hello
            port: {number: 80}
EOF
```

### Verification

```bash
kubectl -n l5-lab get svc,ingress,endpoints
# - hello: ClusterIP, 2 endpoints
# - hello-np: NodePort, 2 endpoints, NodePort assigned
# - hello-ing: HOSTS=hello.local, CLASS=nginx, ADDRESS (after a few seconds)

# Test ingress
INGRESS_IP=$(kubectl -n ingress-nginx get pod \
  -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].status.podIP}')
kubectl run t --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  curl -s -H 'Host: hello.local' http://$INGRESS_IP/
```

Expected output: `hello-from-pod` (from one of the two pods).

### Cleanup

```bash
kubectl delete ns l5-lab
```

---

## Concepts

The **Gateway API** (`gateway.networking.k8s.io`) is the role-oriented successor
to Ingress (the Ingress API is **frozen**). It ships as **CRDs** + a controller.

| Kind | Owned by | Purpose |
|------|----------|---------|
| **GatewayClass** | Infra provider | Selects the controller/implementation (like an IngressClass) |
| **Gateway** | Cluster operator | A traffic entry point — listeners, ports, protocols |
| **HTTPRoute** | App developer | Maps requests (host/path/header) to backend Services |

- A **Gateway** references a **GatewayClass**; routes attach to a Gateway via
  **`parentRefs`** and forward to Services via **`backendRefs`**.
- Route kinds: **HTTPRoute, GRPCRoute, TLSRoute, TCPRoute, UDPRoute** (HTTPRoute
  and GRPCRoute are the stable ones).
- More expressive than Ingress: header/method matching, traffic splitting,
  cross-namespace routing; supports HTTP/HTTPS/TLS/TCP/gRPC.

## Setup — install the Gateway API CRDs

The CRDs are **not** built in; install the standard channel:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
kubectl get crd | grep gateway.networking.k8s.io
```

Sample output:

```
customresourcedefinition.../gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.../gateways.gateway.networking.k8s.io created
customresourcedefinition.../grpcroutes.gateway.networking.k8s.io created
customresourcedefinition.../httproutes.gateway.networking.k8s.io created
customresourcedefinition.../referencegrants.gateway.networking.k8s.io created
```

> For real routing you then install an **implementation**, e.g. NGINX Gateway
> Fabric, which provides a `GatewayClass` (often named `nginx`) and the data plane.

---

## Examples

### 1. A backend Service

```bash
kubectl create ns gw-demo
kubectl -n gw-demo create deployment echo --image=hashicorp/http-echo:1.0 \
  -- /http-echo -listen=:5678 -text="hello-from-gateway-api"
kubectl -n gw-demo expose deployment echo --port=80 --target-port=5678
```

### 2. GatewayClass → Gateway → HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata: {name: example-class}
spec:
  controllerName: example.com/gateway-controller    # a real controller's name
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: {name: example-gateway, namespace: gw-demo}
spec:
  gatewayClassName: example-class
  listeners:
  - {name: http, protocol: HTTP, port: 80}
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: echo-route, namespace: gw-demo}
spec:
  parentRefs:
  - {name: example-gateway}            # attach to the Gateway
  hostnames: ["echo.example.com"]
  rules:
  - matches:
    - path: {type: PathPrefix, value: /}
    backendRefs:
    - {name: echo, port: 80}           # forward to the Service
```

```bash
kubectl apply -f gateway.yaml
kubectl get gatewayclass example-class
kubectl -n gw-demo get gateway,httproute
```

Sample output (with the placeholder controllerName, **no controller running**):

```
NAME            CONTROLLER                       ACCEPTED
example-class   example.com/gateway-controller   Unknown

NAME              CLASS           ADDRESS   PROGRAMMED
example-gateway   example-class             Unknown

NAME         HOSTNAMES
echo-route   ["echo.example.com"]
```

The resources are **created and schema-valid**, but `ACCEPTED`/`PROGRAMMED` are
**`Unknown`** and the Gateway has **no `ADDRESS`** — because no controller claims
`example.com/gateway-controller`. This is the Gateway-API equivalent of *"an
Ingress does nothing without an ingress controller."*

### 3. With a real controller (what changes)

If you install an implementation and point `gatewayClassName` at its GatewayClass
(e.g. `nginx` from NGINX Gateway Fabric):

- `GatewayClass ACCEPTED=True`, `Gateway PROGRAMMED=True`, and the Gateway gets an
  **ADDRESS**.
- A request with `Host: echo.example.com` to that address is routed by the
  `HTTPRoute` to the `echo` Service → a Pod returns `hello-from-gateway-api`.

```bash
# Example test once a controller assigns an address:
ADDR=$(kubectl -n gw-demo get gateway example-gateway -o jsonpath='{.status.addresses[0].value}')
curl -s -H 'Host: echo.example.com' "http://$ADDR/"
```

---

## Lab

**Goal**: install the Gateway API CRDs and model HTTP routing to a Service with a
`GatewayClass` / `Gateway` / `HTTPRoute`.

### Steps

1. Install the Gateway API **standard CRDs**.
2. In namespace `gw-lab`, deploy `site` (`hashicorp/http-echo:1.0`, text
   `routed-by-gateway`, port 5678) and expose it as `site:80`.
3. Create a `GatewayClass` `lab-class`, a `Gateway` `lab-gw` (HTTP listener :80),
   and an `HTTPRoute` `site-route` that routes `Host: site.lab` (prefix `/`) to
   `site:80`.
4. Show the three resources and explain why the `Gateway` has no `ADDRESS`.

### Solution

```bash
# 1
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# 2
kubectl create ns gw-lab
kubectl -n gw-lab create deployment site --image=hashicorp/http-echo:1.0 \
  -- /http-echo -listen=:5678 -text="routed-by-gateway"
kubectl -n gw-lab expose deployment site --port=80 --target-port=5678

# 3
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

# 4
kubectl get gatewayclass lab-class
kubectl -n gw-lab get gateway,httproute
```

### Verification

```bash
kubectl get crd | grep -c gateway.networking.k8s.io     # 5 CRDs
kubectl -n gw-lab get httproute site-route \
  -o jsonpath='{.spec.parentRefs[0].name} -> {.spec.rules[0].backendRefs[0].name}:{.spec.rules[0].backendRefs[0].port}{"\n"}'
# lab-gw -> site:80
```

Expected: the 5 Gateway-API CRDs exist; `GatewayClass`/`Gateway`/`HTTPRoute` are
created; the `HTTPRoute` binds `lab-gw → site:80`. The `Gateway` shows **no
ADDRESS** and `PROGRAMMED=Unknown` because no controller implements
`example.com/gateway-controller` — installing an implementation (NGINX Gateway
Fabric, Istio, Cilium) is what makes it route.

### Cleanup

```bash
kubectl delete ns gw-lab
kubectl delete gatewayclass lab-class
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```
