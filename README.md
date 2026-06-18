# Kubernetes en Production — Administration (CKA) + Déploiement (Kubespray)

Formation combinée **CKA + Kubespray** : une seule progression cohérente qui fusionne
l'administration Kubernetes (parcours CKA) et le déploiement de cluster de production
(Kubespray), sans redondance. La création de cluster est enseignée avec **Kubespray**
(Ansible, production) plutôt que le `kubeadm` manuel — kubeadm reste le moteur sous-jacent.

**24 leçons (0–23)** · supports compilés en **PDF** · **labs en texte (Markdown)**.

## Contenu du dépôt

| Dossier | Contenu |
|---------|---------|
| `pdf/Formation_Combined_Complete.pdf` | Support **complet** (slides + résumés détaillés) |
| `pdf/Formation_Combined_Slides.pdf` | **Slides** seuls (présentation) |
| `pdf/Combined_5day_Delivery_Plan.pdf` | Plan de livraison **5 jours** |
| `pdf/Combined_10day_Delivery_Plan.pdf` | Plan de livraison **10 jours** (hands-on) |
| `pdf/lessons/` | Un **PDF par leçon** (L00–L23) |
| `labs/` | **Travaux pratiques** au format Markdown (énoncés + solutions) |

## Travaux pratiques (`labs/`)

- `setup.md` — **lab fondateur** : le cluster partagé, un Kubernetes multi-nœuds déployé
  sur **OpenStack via Kubespray** (Terraform `contrib/terraform/openstack` → `cluster.yml`,
  cloud-provider externe + Octavia LBaaS + Cinder CSI).
- `Lesson_01..23_*.md` — un lab par leçon (concepts, exemples, énoncé, solution, vérif, cleanup).
- `Combined_Labs_Questions.md` / `Combined_Labs_Answers.md` — recueil énoncés / corrigés.

> ✅ **Labs validés sur un cluster réel.** L'ensemble des labs a été exécuté sur un cluster
> HA (3 control-plane + 2 workers) déployé par Kubespray sur OpenStack, puis adapté aux
> spécificités OpenStack/Kubespray (StorageClass `cinder-csi`, LoadBalancer via Octavia,
> ingress-nginx en mode cloud, etc.) — ces écarts par rapport à un cluster `kind` sont
> signalés directement dans chaque lab.

## Cursus (24 leçons)

| # | Leçon | # | Leçon |
|---|-------|---|-------|
| 00 | Introduction | 12 | Scheduling |
| 01 | Architecture Kubernetes | 13 | Networking & NetworkPolicy |
| 02 | Requirements & Environnement | 14 | Security (RBAC) |
| 03 | Installation & Inventory (Kubespray) | 15 | CRDs & Operators |
| 04 | Configuration (`group_vars`) | 16 | Node Maintenance & Scaling |
| 05 | Container Runtimes & CNI | 17 | Upgrades |
| 06 | Déploiement du cluster (`cluster.yml`) | 18 | Backup, Recovery & Reset |
| 07 | High Availability | 19 | Add-ons & Packaging (Helm/Kustomize) |
| 08 | Workloads & Autoscaling (HPA) | 20 | Logging, Monitoring & Troubleshooting |
| 09 | Storage (PV/PVC/StorageClass) | 21 | Hardening & Offline |
| 10 | Application Access (Services/Ingress/Gateway) | 22 | Cloud & OpenStack |
| 11 | Configuration & Quotas | 23 | Exam Preparation & Wrap-up |

## Construction → usage → opérations Day-2 → avancé

```
Construire (Kubespray)  →  Utiliser (CKA app/admin)  →  Opérer Day-2  →  Avancé
   L00–L07                    L08–L15                     L16–L20         L21–L23
```

---
*Sources : documentation officielle `kubernetes.io/docs`, `kubespray.io` /
`kubernetes-sigs/kubespray`, `helm.sh`. Slides Marp, supports md-to-pdf.*
