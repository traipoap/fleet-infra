# Fleet Infra — GitOps Platform

> GitOps repository for a K3s cluster on Proxmox. All infrastructure, observability, security, and application workloads are declared in Git and continuously reconciled by Flux CD.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-326ce5)]()
[![GitOps](https://img.shields.io/badge/GitOps-FluxCD-5468ff)]()
[![Istio](https://img.shields.io/badge/Istio-Gateway%20API-467bbf)]()
[![cert-manager](https://img.shields.io/badge/cert-manager-ACME-4aa8ff)]()
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

> 📖 Full platform documentation (Terraform, Ansible, CI/CD, network topology) → [README-platform.md](README-platform.md)

---

## Table of Contents

- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Repository Layout](#repository-layout)
- [Flux CD Reconciliation Flow](#flux-cd-reconciliation-flow)
- [Infrastructure Components](#infrastructure-components)
  - [Security — cert-manager](#security--cert-manager)
  - [Observability](#observability)
  - [Logging](#logging)
  - [Networking & Storage](#networking--storage)
- [Application Deployment](#application-deployment)
- [Cluster Targets](#cluster-targets)
- [Verification Commands](#verification-commands)
- [Maintenance Checklist](#maintenance-checklist)

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Git (Source of Truth)                               │
│                     github.com/traipoap/fleet-infra                     │
└──────────────────────────────┬─────────────────────────────────────────┘
                               │ push
                               ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     Flux CD (flux-system namespace)                     │
│                                                                        │
│  GitRepository: flux-system                                            │
│       │                                                                │
│       ▼                                                                │
│  Kustomization: flux-system  ──►  clusters/dev/sources/                │
│       │                                                                │
│       ├──► Kustomization: cert-manager          (Helm: cert-manager)   │
│       │         │                                                      │
│       │         ▼                                                      │
│       ├──► Kustomization: cert-manager-config   (ClusterIssuers)       │
│       │         │                                                      │
│       │         ▼                                                      │
│       ├──► Kustomization: app                 (targetNS: lumina)       │
│       │                                                                │
│       ├──► Kustomization: infra              (logging, observability,  │
│       │                                    networking, security)       │
│       │                                                                │
│       └──► HelmReleases: vector, quickwit, nfs, weave-gitops           │
└──────────────────────────────┬─────────────────────────────────────────┘
                               │ reconcile
                               ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     K3s Cluster (HA, embedded etcd)                     │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  cert-manager (cert-manager NS)                                  │   │
│  │  └── ClusterIssuers: letsencrypt-prod, selfsigned-issuer         │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  Istio (istio-system NS)                                         │   │
│  │  ├── Gateway + HTTPRoute (frontend.codezap.win)                  │   │
│  │  ├── Prometheus ──► Grafana ──► Kiali                            │   │
│  │  └── Service mesh (mTLS, traffic policies)                       │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  Logging (lumina NS)                                             │   │
│  │  ├── Vector agent (syslog + k8s logs)                            │   │
│  │  └── Quickwit (indexer + searcher)                               │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  Storage                                                         │   │
│  │  ├── NFS provisioner (RWX PersistentVolumes)                     │   │
│  │  └── Garage S3 (object storage, super-node)                      │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  Application (lumina NS)                                         │   │
│  │  ├── Frontend (Astro, port 4321)                                 │   │
│  │  └── Backend (Go API, port 8080)                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Start

### Prerequisites

| Tool | Min Version | Purpose |
|------|-------------|---------|
| `flux` | ≥ 2.3 | Bootstrap & manage GitOps |
| `kubectl` | ≥ 1.28 | Cluster access |
| `kustomize` | ≥ 5.0 | Local preview / diff |
| `helm` | ≥ 3.12 | Chart inspection |

### Cluster Requirements

- K3s (HA, embedded etcd)
- Istio installed (Gateway API class `istio`)
- Namespace `istio-system` exists

### Bootstrap

```bash
flux bootstrap github \
  --owner=traipoap \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/dev \
  --personal
```

### Verify

```bash
flux get all -A
kubectl get pods -A
```

---

## Repository Layout

```
fleet-infra/
├── app/                              # Application manifests
│   └── base/
│       ├── kustomization.yaml
│       ├── namespace.yaml            # lumina NS (Istio ambient mode)
│       ├── configmap.yaml            # Shared env config
│       ├── frontend-deployment.yaml  # Astro app (port 4321)
│       ├── frontend-service.yaml
│       ├── backend-deployment.yaml   # Go API (port 8080)
│       ├── backend-service.yaml
│       ├── backend-pvc.yaml          # RWX PVC (NFS)
│       ├── gateway.yaml              # Istio Gateway (HTTP+HTTPS, TLS)
│       ├── httproute.yaml            # Path-based routing
│       ├── cert-lumina.yaml          # Certificate (cert-manager)
│       └── ingress.yaml              # Legacy fallback
│
├── clusters/                         # Per-cluster GitOps definitions
│   └── dev/                          # Dev environment
│       ├── flux-system/              # Flux bootstrap
│       │   ├── gotk-components.yaml  #   Flux CRDs
│       │   ├── gotk-sync.yaml        #   → path: ./clusters/dev/sources
│       │   └── kustomization.yaml
│       │
│       ├── infra/                    # Platform components (Kustomize)
│       │   ├── kustomization.yaml    #   → logging, networking, observability, security
│       │   │
│       │   ├── security/
│       │   │   ├── kustomization.yaml
│       │   │   └── cert-manager/
│       │   │       ├── kustomization.yaml
│       │   │       └── cert-manager-config.yaml  # ClusterIssuers
│       │   │
│       │   ├── observability/
│       │   │   ├── kustomization.yaml
│       │   │   ├── prometheus/kustomization.yaml  # Istio Prometheus addon
│       │   │   ├── grafana/kustomization.yaml     # Istio Grafana addon
│       │   │   └── kiali/kustomization.yaml       # Istio Kiali addon
│       │   │
│       │   ├── logging/
│       │   │   ├── kustomization.yaml
│       │   │   ├── otel-config.yaml               # OTel Collector (comparison)
│       │   │   ├── vector/
│       │   │   │   ├── kustomization.yaml
│       │   │   │   ├── values.yaml                #   Remap pipeline
│       │   │   │   └── vector-release.yaml        #   HelmRelease
│       │   │   └── quickwit/
│       │   │       ├── kustomization.yaml
│       │   │       ├── quickwit-release.yaml      #   Indexer + Searcher
│       │   │       ├── quickwit-indexer.yaml
│       │   │       └── job-create-index.yaml      #   Index schema job
│       │   │
│       │   └── networking/
│       │       ├── kustomization.yaml
│       │       └── nfs/
│       │           ├── kustomization.yaml
│       │           └── nfs-subdir-external-provisioner-release.yaml
│       │
│       └── sources/                  # Flux CRs (Kustomizations + HelmReleases)
│           ├── kustomization.yaml    #   Top-level: aggregates all below
│           ├── cert-manager-repo.yaml #  HelmRepo + HelmRelease + Kustomization
│           ├── vector-repo.yaml      #   HelmRepo + HelmRelease
│           ├── quickwit-repo.yaml    #   HelmRepo + HelmRelease
│           ├── nfs-repo.yaml         #   HelmRepo + HelmRelease
│           ├── weave-gitops-dashboard.yaml  # HelmRepo + HelmRelease (UI)
│           ├── infra.yaml            #   Kustomization → ./clusters/dev/infra
│           └── app.yaml              #   Kustomization → ./app/base
│
├── README.md                         # This file (GitOps repo)
├── README-platform.md                # Full platform docs (Terraform, Ansible, CI/CD)
└── .gitignore
```

---

## Flux CD Reconciliation Flow

### Dependency Chain

```
flux-system (Kustomization)
  │
  │  applies: clusters/dev/sources/
  │
  ├──► cert-manager-repo.yaml
  │     ├── HelmRepository: jetstack          (oci://quay.io/jetstack/charts)
  │     ├── HelmRelease: cert-manager         (targetNS: cert-manager, installCRDs: true)
  │     └── Kustomization: cert-manager-config  ← dependsOn: cert-manager
  │           path: ./clusters/dev/infra/security/cert-manager
  │
  ├──► vector-repo.yaml          → HelmRelease: vector
  ├──► quickwit-repo.yaml        → HelmRelease: quickwit
  ├──► nfs-repo.yaml             → HelmRelease: nfs-subdir-external-provisioner
  ├──► weave-gitops-dashboard.yaml → HelmRelease: ww-gitops
  │
  ├──► infra.yaml
  │     Kustomization: infra
  │       path: ./clusters/dev/infra
  │       wait: true
  │       │
  │       ├──► security/cert-manager/   (ClusterIssuers)
  │       ├──► observability/           (Prometheus, Grafana, Kiali)
  │       ├──► logging/                 (Vector, Quickwit)
  │       └──► networking/nfs/          (NFS provisioner)
  │
  └──► app.yaml
        Kustomization: app
          dependsOn: cert-manager-config
          path: ./app/base
          targetNamespace: lumina
          wait: true
```

### Reconciliation Order

| # | Resource | Type | Namespace | Waits For |
|---|----------|------|-----------|-----------|
| 1 | `jetstack` | HelmRepository | flux-system | — |
| 2 | `cert-manager` | HelmRelease | flux-system | jetstack |
| 3 | `cert-manager-config` | Kustomization | flux-system | cert-manager |
| 4 | `infra` | Kustomization | flux-system | — |
| 5 | `app` | Kustomization | flux-system | cert-manager-config |
| 6 | `vector` | HelmRelease | flux-system | — |
| 7 | `quickwit` | HelmRelease | flux-system | — |
| 8 | `nfs-subdir-external-provisioner` | HelmRelease | flux-system | — |
| 9 | `ww-gitops` | HelmRelease | flux-system | — |

---

## Infrastructure Components

### Security — cert-manager

| Component | Source | Notes |
|-----------|--------|-------|
| cert-manager CRDs + controller | Helm chart (OCI: `oci://quay.io/jetstack/charts`) | `installCRDs: true` |
| `letsencrypt-prod` ClusterIssuer | `clusters/dev/infra/security/cert-manager/cert-manager-config.yaml` | ACME prod, HTTP-01 via Istio |
| `selfsigned-issuer` ClusterIssuer | same file | Dev / testing |

**Dependency:** `cert-manager-config` Kustomization has `dependsOn: cert-manager` HelmRelease — ensures CRDs are registered before ClusterIssuers are applied.

The Gateway references the issuer:
```yaml
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

### Observability

| Component | Namespace | Source |
|-----------|-----------|--------|
| Prometheus | `istio-system` | Istio addon manifest (remote URL) |
| Grafana | `istio-system` | Istio addon + Quickwit datasource patch |
| Kiali | `istio-system` | Istio addon manifest (remote URL) |

**Grafana plugins:** Quickwit datasource auto-installed via `GF_INSTALL_PLUGINS`.

**Datasources configured:**
- Prometheus → `http://prometheus:9090`
- Quickwit → `http://quickwit-searcher.lumina.svc.cluster.local:7280/api/v1`

### Logging

| Component | Role | Sink |
|-----------|------|------|
| **Vector** (primary) | DaemonSet agent — collects syslog (TCP/UDP:514) + k8s pod logs | HTTP POST → Quickwit `:7280` |
| **OTel Collector** (comparison) | DaemonSet — reads `/var/log/pods/**/*.log` | OTLP → Quickwit `:7281` |
| **Quickwit** | Indexer + Searcher | Stores & serves log queries |

**Vector pipeline:**
```
syslog_tcp/udp ─┐
kubernetes_logs ─┼─► remap (.index_timestamp) ─► HTTP sink → Quickwit
                 ┘
```

### Networking & Storage

| Component | Purpose |
|-----------|---------|
| NFS Subdir External Provisioner | RWX PersistentVolumes (NFS server: `YOUR_INFRA_IP`) |
| Garage S3 (super-node) | Object storage (backups, registry backend, app assets) |

---

## Application Deployment

| Service | Image | Port | Resources |
|---------|-------|------|-----------|
| Frontend (Astro) | `ghcr.io/traipoap/frontend` | 4321 | Req: 64Mi/50m · Lim: 128Mi/200m |
| Backend (Go) | `ghcr.io/traipoap/backend` | 8080 | Req: 64Mi/50m · Lim: 256Mi/200m |

### Routing (`app/base/httproute.yaml`)

```
frontend.codezap.win/
    ├── /*              → frontend-svc:4321   (static site)
    ├── /api/auth/login → backend-svc:8080    (auth)
    ├── /api/indices    → backend-svc:8080    (index mgmt)
    └── /api/search     → backend-svc:8080    (search → Quickwit)
```

All routes terminate at the **Istio Gateway** with TLS via cert-manager.

---

## Cluster Targets

| Cluster | Path | Purpose |
|---------|------|---------|
| Dev | `clusters/dev/` | Development (K3s HA on Proxmox) |

To add a new environment:

```bash
# 1. Copy the dev structure
cp -r clusters/dev clusters/staging

# 2. Bootstrap Flux on the new cluster
flux bootstrap github \
  --owner=traipoap \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/staging \
  --personal
```

---

## Verification Commands

```bash
# Flux status
flux get all -A
flux get kustomizations -n flux-system
flux get helmreleases -n flux-system

# Force reconcile
flux reconcile kustomization flux-system -n flux-system
flux reconcile kustomization app -n flux-system --with-source

# Cluster health
kubectl get nodes -o wide
kubectl get pods -A
kubectl get gateway -n lumina
kubectl get httproute -n lumina
kubectl get certificate -n lumina

# Observability
kubectl -n istio-system port-forward svc/prometheus-server 9090:9090
kubectl -n istio-system port-forward svc/grafana 3000:3000
kubectl -n istio-system port-forward svc/kiali 20001:20001

# Logging
kubectl -n lumina logs -l app.kubernetes.io/name=vector
kubectl -n lumina port-forward svc/quickwit-searcher 7280:7280
```

---

## Maintenance Checklist

- [ ] Pin Helm chart versions (currently using latest)
- [ ] Add `PodDisruptionBudget` for infra components before production
- [ ] Configure Flux `Alert` + `Receiver` (Slack/Email) for reconciliation failures
- [ ] Add NetworkPolicies for namespace isolation
- [ ] Add Kyverno/OPA Gatekeeper policy enforcement
- [ ] Replace remote Istio addon URLs with local rendered manifests
- [ ] Add Velero backup for etcd + PVs
- [ ] Multi-environment promotion pipeline (dev → staging → prod)
