# Fleet Infra — GitOps Repository

> GitOps source of truth for a K3s Kubernetes cluster. Managed by **Flux CD** — all infrastructure and application state is declared in Git and continuously reconciled.

Part of the [gitops-platform](https://github.com/traipoap/gitops-platform) project. This repo contains only the Kubernetes manifests and Flux CRs; Terraform, Ansible, CI/CD, and application code live in the main repo.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Git (this repo — main branch)                   │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │  flux watches (1m interval)
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Flux CD (flux-system namespace)                      │
│                                                                         │
│  GitRepository: flux-system                                             │
│       │                                                                 │
│       ▼                                                                 │
│  Kustomization: flux-system  ──►  clusters/dev/sources/                 │
│       │                                                                 │
│       ├── HelmRepository: jetstack ──► HelmRelease: cert-manager        │
│       ├── HelmRepository: vector-repo ──► HelmRelease: vector           │
│       ├── HelmRepository: quickwit-repo ──► HelmRelease: quickwit       │
│       ├── HelmRepository: nfs-subdir-external-provisioner ──► HelmRelease: nfs-subdir-external-provisioner │
│       ├── HelmRepository: ww-gitops ──► HelmRelease: ww-gitops          │
│       │                                                                 │
│       ▼  (dependency chain — each waits for the previous)               │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ nfs-subdir-external-provisioner  (NFS StorageClass)               │  │
│  │       └► quickwit  (log store: indexer + searcher)                │  │
│  │              └► vector  (log shipper: k8s + syslog → Quickwit)    │  │
│  │                     └► observability  (observability + security)  │  │
│  │                            └► cert-manager-config  (CA Issuer)    │  │
│  │                                   └► app  (frontend + backend)    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │  reconcile
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        K3s Cluster (HA, 3 masters)                      │
│                                                                         │
│  istio-system/     Gateway API, Prometheus, Grafana, Kiali              │
│  cert-manager/     CA Issuer (self-signed)                              │
│  lumina/           App: frontend (Astro), backend (Go), Gateway,        │
│                    HTTPRoute, Vector, Quickwit                          │
│  nfs/              NFS Subdir External Provisioner                      │
│  flux-system/      Flux controllers, Weave GitOps Dashboard             │
└─────────────────────────────────────────────────────────────────────────┘
```

### Dependency Chain

Each Kustomization **blocks** until the previous one is `Ready`:

```
flux-system (bootstrap)
  └─► nfs-subdir-external-provisioner    ← NFS StorageClass for Quickwit PVCs
        └─► quickwit                     ← log store (indexer + searcher)
              └─► vector                 ← log shipper (needs Quickwit endpoint)
                    └─► observability    ← observability (Prometheus, Grafana, Kiali)
                          └─► cert-manager-config   ← CA Issuer (TLS for Gateway)
                                └─► app             ← frontend + backend + Gateway + HTTPRoute
```

---

## Repository Structure

```
.
├── app/
│   └── base/
│       ├── kustomization.yaml          # Bundles all app resources
│       ├── namespace.yaml              # lumina (Istio ambient mode)
│       ├── configmap.yaml              # Shared env config
│       ├── frontend-deployment.yaml    # Astro frontend (port 4321)
│       ├── frontend-service.yaml       # ClusterIP → 4321
│       ├── backend-deployment.yaml     # Go API server (port 8080)
│       ├── backend-service.yaml        # ClusterIP → 8080
│       ├── backend-pvc.yaml            # Persistent storage for exports
│       ├── gateway.yaml                # Istio Gateway (HTTP + HTTPS, TLS)
│       ├── httproute.yaml              # Path-based routing rules
│       ├── ingress.yaml                # Legacy fallback
│       └── cert-lumina.yaml            # Certificate resource
│
├── clusters/
│   └── dev/
│       ├── flux-system/
│       │   ├── kustomization.yaml
│       │   ├── gotk-components.yaml    # Flux CRDs + controllers
│       │   └── gotk-sync.yaml          # GitRepository + Kustomization (→ ./clusters/dev/sources)
│       │
│       ├── infra/
│       │   ├── kustomization.yaml      # Aggregates: infra (+ logging, networking, security)
│       │   ├── observability/
│       │   │   ├── kustomization.yaml
│       │   │   ├── prometheus/kustomization.yaml   # Istio Prometheus addon
│       │   │   ├── grafana/kustomization.yaml      # Istio Grafana addon + Quickwit datasource
│       │   │   └── kiali/kustomization.yaml        # Istio Kiali addon
│       │   ├── logging/
│       │   │   ├── kustomization.yaml
│       │   │   ├── otel-config.yaml                # OTel Collector (alternative pipeline)
│       │   │   ├── vector/
│       │   │   │   ├── kustomization.yaml
│       │   │   │   ├── values.yaml                 # Custom remap → Quickwit HTTP sink
│       │   │   │   └── vector-release.yaml         # HelmRelease
│       │   │   └── quickwit/
│       │   │       ├── kustomization.yaml
│       │   │       ├── quickwit-release.yaml       # HelmRelease (indexer + searcher)
│       │   │       ├── quickwit-indexer.yaml
│       │   │       └── job-create-index.yaml       # Index schema job
│       │   ├── networking/
│       │   │   ├── kustomization.yaml
│       │   │   └── nfs/
│       │   │       ├── kustomization.yaml
│       │   │       └── nfs-subdir-external-provisioner-release.yaml
│       │   └── security/
│       │       ├── kustomization.yaml
│       │       └── cert-manager/
│       │           ├── kustomization.yaml
│       │           ├── ca-issuer.yaml              # Self-signed CA ClusterIssuer
│       │           ├── letsencrypt-http01.yaml     # Let's Encrypt HTTP-01 (for prod)
│       │           ├── letsencrypt-dns01-cloudflare.yaml  # Let's Encrypt DNS-01
│       │           ├── self-signed-issuer.yaml     # Self-signed (dev)
│       │           └── wildcard-cert.yaml          # Wildcard cert
│       │
│       └── sources/
│           ├── kustomization.yaml          # Top-level: all Flux CRs
│           ├── app.yaml                    # Kustomization: app (dependsOn: cert-manager-config)
│           ├── observability.yaml          # Kustomization: observability (dependsOn: vector)
│           ├── cert-manager-repo.yaml      # Namespace + HelmRepo + HelmRelease + Kustomization
│           ├── vector-repo.yaml            # HelmRepo + Kustomization: vector (dependsOn: quickwit)
│           ├── quickwit-repo.yaml          # Namespace + HelmRepo + Kustomization: quickwit (dependsOn: nfs)
│           ├── nfs-repo.yaml               # Namespace + HelmRepo + Kustomization: nfs (dependsOn: flux-system)
│           └── weave-gitops-dashboard.yaml # HelmRepo + HelmRelease: Weave GitOps UI
│
└── README.md
```

---

## Flux CD Resources

### Sources

| Kind | Name | Namespace | Purpose |
|------|------|-----------|---------|
| `GitRepository` | `flux-system` | `flux-system` | Points to this repo (main branch) |
| `HelmRepository` | `jetstack` | `flux-system` | cert-manager OCI charts (`quay.io/jetstack/charts`) |
| `HelmRepository` | `vector-repo` | `lumina` | Vector charts (`helm.vector.dev`) |
| `HelmRepository` | `quickwit-repo` | `lumina` | Quickwit charts (`helm.quickwit.io`) |
| `HelmRepository` | `nfs-subdir-external-provisioner` | `nfs` | NFS provisioner charts |
| `HelmRepository` | `ww-gitops` | `flux-system` | Weave GitOps OCI charts |

### Kustomizations (dependency order)

| # | Name | Path | `dependsOn` | Target NS |
|---|------|------|-------------|-----------|
| 1 | `flux-system` | `./clusters/dev/sources` | — | — |
| 2 | `nfs-subdir-external-provisioner` | `./clusters/dev/infra/networking/nfs` | `flux-system` | `nfs` |
| 3 | `quickwit` | `./clusters/dev/infra/logging/quickwit` | `nfs-subdir-external-provisioner` | `lumina` |
| 4 | `vector` | `./clusters/dev/infra/logging/vector` | `quickwit` | `lumina` |
| 5 | `infra` | `./clusters/dev/infra` | `vector` | — |
| 6 | `cert-manager-config` | `./clusters/dev/infra/security/cert-manager` | `infra` | — |
| 7 | `app` | `./app/base` | `cert-manager-config` | `lumina` |

### HelmReleases

| Name | Namespace | Chart | Source |
|------|-----------|-------|--------|
| `cert-manager` | `flux-system` | `cert-manager` (jetstack) | `jetstack` |
| `ww-gitops` | `flux-system` | `weave-gitops` | `ww-gitops` |

> Vector and Quickwit are deployed via Kustomizations (not HelmReleases) that reference their HelmRepositories.

---

## Application

### Routing

```
frontend.codezap.win
  ├── /*                  → frontend-svc:4321   (Astro static site)
  ├── /api/auth/login     → backend-svc:8080    (auth)
  ├── /api/auth/refresh   → backend-svc:8080    (token refresh)
  ├── /api/indices        → backend-svc:8080    (index management)
  ├── /api/search         → backend-svc:8080    (log search)
  ├── /api/export         → backend-svc:8080    (log export)
  └── /api/exports        → backend-svc:8080    (export list)
```

### TLS

- **Gateway** terminates TLS on port 443
- **ClusterIssuer**: `cluster-ca` (self-signed CA)
- Certificate: `frontend-codezap-win-tls`
- Gateway API class: `istio`

---

## Observability

| Component | Namespace | Access |
|-----------|-----------|--------|
| Prometheus | `istio-system` | `kubectl -n istio-system port-forward svc/prometheus-server 9090:9090` |
| Grafana | `istio-system` | `kubectl -n istio-system port-forward svc/grafana 3000:3000` |
| Kiali | `istio-system` | `kubectl -n istio-system port-forward svc/kiali 20001:20001` |
| Quickwit | `lumina` | `kubectl -n lumina port-forward svc/quickwit-searcher 7280:7280` |
| Weave GitOps | `flux-system` | `kubectl -n flux-system port-forward svc/ww-gitops 8080:8080` |

### Logging Pipeline

```
Pod logs ──► Vector (k8s_logs + syslog) ──► Quickwit (HTTP sink)
                                                    │
                                                    ▼
                                            Grafana (Quickwit datasource)
```

---

## Quick Start

### Bootstrap Flux

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
kubectl get kustomization -n flux-system
kubectl get helmrelease -n flux-system
kubectl get pods -A
```

### Force Reconcile

```bash
flux reconcile kustomization flux-system -n flux-system --with-source
flux reconcile kustomization <name> -n flux-system --with-source
```

---

## Adding a New Environment

To deploy to a second cluster (e.g., `staging`):

```bash
# 1. Copy the dev structure
cp -r clusters/dev clusters/staging

# 2. Update the git URL in gotk-sync.yaml (or use a different repo)

# 3. Bootstrap Flux on the new cluster
flux bootstrap github \
  --owner=traipoap \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/staging \
  --personal
```

Each cluster has its own `sources/` directory with independent Flux CRs.

---

## Maintenance Checklist

- [ ] Pin Helm chart versions (currently using latest)
- [ ] Switch from self-signed CA to Let's Encrypt for production domains
- [ ] Add PodDisruptionBudgets for stateful components (Quickwit, Vector)
- [ ] Add NetworkPolicies for namespace isolation
- [ ] Configure Flux alerts (Slack/Email) via `Alert` resources
- [ ] Add Kyverno/OPA policies for security enforcement
- [ ] Multi-environment promotion (dev → staging → prod)
