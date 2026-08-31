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
│  ArtifactGenerator: flux-system                                         │
│       │  packages into:                                                 │
│       │  ├── infrastructure  (infrastructure/**)                        │
│       │  └── apps            (apps/base/** + apps/staging/**)           │
│       │                                                                 │
│       ▼                                                                 │
│  Kustomization: infra-controllers  ──►  ./controllers                   │
│  Kustomization: infra-configs      ──►  ./configs                       │
│  Kustomization: apps               ──►  ./staging                       │
│                                                                         │
│  Controllers (via HelmReleases):                                        │
│  ├── cert-manager          (jetstack OCI)                               │
│  ├── external-secrets      (AWS SecretsManager)                         │
│  ├── vector                (log shipper)                                │
│  ├── quickwit              (log store)                                  │
│  ├── nfs-subdir-external   (NFS StorageClass)                           │
│  └── weave-gitops          (GitOps dashboard)                           │
│                                                                         │
│  Image Automation (ImageUpdateAutomation):                              │
│  ├── backend   (ghcr.io/traipoap/backend → apps/staging)                │
│  └── frontend  (ghcr.io/traipoap/frontend → apps/staging)               │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │  reconcile
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        K3s Cluster (HA, 3 masters)                      │
│                                                                         │
│  istio-system/       Gateway API, Prometheus, Grafana, Kiali            │
│  cert-manager/       CA Issuer (self-signed)                            │
│  external-secrets/   External Secrets Operator + ClusterSecretStore     │
│  lumina/             App: frontend, backend, Gateway, HTTPRoute         │
│                      Vector, Quickwit, Image Automation CRDs            │
│  nfs/                NFS Subdir External Provisioner                    │
│  flux-system/        Flux controllers, Weave GitOps Dashboard           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Design: ArtifactGenerator + ExternalArtifact

Flux's **ArtifactGenerator** packages subdirectories into artifacts, and Kustomizations reference them via `ExternalArtifact` source kind. This decouples the manifest layout from the Flux reconciliation path:

```
infrastructure/**  ──►  Artifact: infrastructure  ──►  Kustomization: infra-controllers
                                              └─────►  Kustomization: infra-configs
apps/base/**       ──►  Artifact: apps          ──►  Kustomization: apps
apps/staging/**    ──┘
```

This allows:
- **Shared manifests** in `apps/base/` (deployments, services, gateway)
- **Environment overlays** in `apps/staging/` (patches, resource overrides)
- **Clean separation** of infrastructure controllers (Helm) from configs (plain YAML)
- **Image Automation** updating `apps/staging/` without touching `apps/base/`

---

## Repository Structure

```
.
├── apps/                            # Application manifests (shared + per-env overlays)
│   ├── base/                        # Shared: deployments, services, gateway, routing
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml           # lumina (Istio ambient mode)
│   │   ├── configmap.yaml           # Shared env config
│   │   ├── backend-deployment.yaml  # Go API server (port 8080)
│   │   ├── backend-service.yaml     # ClusterIP → 8080
│   │   ├── backend-pvc.yaml         # Persistent storage
│   │   ├── frontend-deployment.yaml # Astro frontend (port 4321)
│   │   ├── frontend-service.yaml    # ClusterIP → 4321
│   │   ├── gateway.yaml             # Istio Gateway (HTTP + HTTPS, TLS)
│   │   ├── httproute.yaml           # Path-based routing rules
│   │   └── ingress.yaml             # Legacy fallback
│   └── staging/                     # Staging overlay (patches, image tags)
│       └── kustomization.yaml       # Extends ../base
│
├── clusters/
│   └── staging/                     # Per-cluster Flux CRs
│       ├── flux-system/             # Flux bootstrap
│       │   └── gotk-sync.yaml       # GitRepository + Kustomization (→ ./clusters/staging)
│       ├── apps.yaml                # Kustomization: apps (ExternalArtifact → ./staging)
│       ├── infrastructure.yaml      # Kustomizations: infra-controllers + infra-configs
│       └── artifacts.yaml           # ArtifactGenerator (packages infra + apps)
│
├── infrastructure/                  # Platform components
│   ├── kustomization.yaml           # Aggregates: configs + controllers
│   ├── configs/                     # Plain YAML configs (no Helm)
│   │   ├── kustomization.yaml
│   │   ├── cert-manager/            # ClusterIssuers (CA, Let's Encrypt, wildcard)
│   │   │   ├── kustomization.yaml
│   │   │   ├── ca-issuer.yaml
│   │   │   ├── letsencrypt-dns01-cloudflare.yaml
│   │   │   ├── letsencrypt-http01.yaml
│   │   │   └── wildcard-cert.yaml
│   │   └── external-secrets/        # External Secrets + AWS SecretsManager
│   │       ├── aws-secret-store.yaml    # ClusterSecretStore (AWS SM)
│   │       ├── jwt-external-secret.yaml # JWT → K8s Secret
│   │       ├── kustomization.yaml
│   │       ├── nfs-external-secret.yaml # NFS credentials
│   │       ├── quickwit-external-secret.yaml  # Quickwit credentials
│   │       ├── registry-external-secret.yaml  # Registry creds → K8s Secret
│   │       └── weave-external-secret.yaml  # Weave credentials
│   │   
│   └── controllers/                 # Helm-managed controllers
│       ├── kustomization.yaml       # Aggregates: automation, logging, networking, observability, security
│       ├── automation/              # Flux Image Automation
│       │   ├── kustomization.yaml
│       │   ├── backend-registry.yaml   # ImageRepository
│       │   ├── backend-policy.yaml     # ImagePolicy (semver)
│       │   ├── backend-automation.yaml # ImageUpdateAutomation
│       │   ├── frontend-registry.yaml
│       │   ├── frontend-policy.yaml
│       │   └── frontend-automation.yaml
│       ├── logging/                 # Log pipeline
│       │   ├── kustomization.yaml
│       │   ├── otel-config.yaml         # OTel Collector (alternative)
│       │   ├── vector/              # Vector (log shipper)
│       │   └── quickwit/            # Quickwit (log store)
│       ├── networking/              # Storage
│       │   ├── kustomization.yaml
│       │   └── nfs-subdir-external-provisioner.yaml
│       ├── observability/           # Monitoring
│       │   ├── kustomization.yaml
│       │   ├── weave-gitops.yaml      # Weave GitOps dashboard
│       │   ├── prometheus/          # Istio Prometheus addon
│       │   ├── grafana/             # Istio Grafana addon
│       │   └── kiali/               # Istio Kiali addon
│       └── security/                # Security controllers
│           ├── kustomization.yaml
│           ├── cert-manager.yaml    # HelmRelease (jetstack OCI)
│           └── external-secrets.yaml # HelmRelease (external-secrets.io)
│
├── scripts/
│   └── validate.sh                  # Manifest validation
│
├── limitrange.yaml                  # Cluster-wide LimitRange
├── require-requests.yaml            # Resource request requirements
└── README.md
```

---

## Flux CD Resources

### Sources

| Kind | Name | Namespace | Purpose |
|------|------|-----------|---------|
| `GitRepository` | `flux-system` | `flux-system` | Points to this repo (main branch) |
| `ArtifactGenerator` | `flux-system` | `flux-system` | Packages `infrastructure/**` and `apps/**` into artifacts |

### Artifacts (via ArtifactGenerator)

| Artifact | Source | Contains |
|----------|--------|----------|
| `infrastructure` | `@monorepo/infrastructure/**` | Controllers + configs manifests |
| `apps` | `@monorepo/apps/base/**` + `@monorepo/apps/staging/**` | Application manifests |

### Kustomizations (apply order)

| # | Name | Source | Path | Purpose |
|---|------|--------|------|---------|
| 1 | `flux-system` | `GitRepository` | `./clusters/staging` | Bootstrap Flux + ArtifactGenerator |
| 2 | `infra-controllers` | `ExternalArtifact` | `./controllers` | HelmReleases: cert-manager, external-secrets, vector, quickwit, nfs, weave, image automation |
| 3 | `infra-configs` | `ExternalArtifact` | `./configs` | ClusterIssuers, ExternalSecrets |
| 4 | `apps` | `ExternalArtifact` | `./staging` | App deployments + overlays |

### HelmReleases

| Name | Namespace | Chart | Source |
|------|-----------|-------|--------|
| `cert-manager` | `cert-manager` | `cert-manager` (jetstack OCI) | `jetstack` |
| `external-secrets` | `external-secrets` | `external-secrets` | `external-secrets` |
| `ww-gitops` | `flux-system` | `weave-gitops` | `ww-gitops` |

### Image Automation (ImageUpdateAutomation)

| Name | Image | Policy | Update Path | Strategy |
|------|-------|--------|-------------|----------|
| `backend` | `ghcr.io/traipoap/backend` | semver `>=0.0.0` | `./apps/staging` | Setters |
| `frontend` | `ghcr.io/traipoap/frontend` | semver `>=0.0.0` | `./apps/staging` | Setters |

> Flux auto-updates image tags in `apps/staging/` and commits as `fluxcdbot`.

### External Secrets

| Kind | Name | Source | Purpose |
|------|------|--------|---------|
| `ClusterSecretStore` | `secretstore-aws` | AWS SecretsManager | Sync remote secrets → K8s |
| `ExternalSecret` | `jwt-secret` | AWS SM | JWT_SECRET for backend |
| `ExternalSecret` | `registry-credentials` | AWS SM | GHCR image pull creds |

---

## Application

### Layout

```
apps/base/        ← shared manifests (deployments, services, gateway)
apps/staging/     ← staging overlay (kustomize patches, image tags)
                     └── kustomization.yaml → extends ../base
```

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

### Secrets (via External Secrets)

- **JWT_SECRET**: Synced from AWS SecretsManager → K8s Secret `jwt-secret` → injected as env var
- **Registry credentials**: Synced from AWS SecretsManager → K8s Secret `github-registry` → `imagePullSecrets`

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

### Image Automation Flow

```
ghcr.io/traipoap/backend:0.0.47 pushed
    │
    ▼
ImageRepository (poll every 5m)
    │
    ▼
ImagePolicy (semver: pick highest ≥ 0.0.0)
    │
    ▼
ImageUpdateAutomation (update tag in apps/staging/ → commit as fluxcdbot → push)
    │
    ▼
Flux Kustomization: apps (detects diff → deploys new image)
```

---

## Quick Start

### Bootstrap Flux

```bash
flux bootstrap github \
  --owner=traipoap \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/staging \
  --personal
```

### Verify

```bash
flux get all -A
kubectl get kustomization -n flux-system
kubectl get helmrelease -n flux-system
kubectl get artifacts -n flux-system
kubectl get imageautomations -n lumina
kubectl get externalsecrets -n lumina
kubectl get pods -A
```

### Force Reconcile

```bash
flux reconcile kustomization flux-system -n flux-system --with-source
flux reconcile kustomization infra-controllers -n flux-system --with-source
flux reconcile kustomization apps -n flux-system --with-source
```

### Validate Manifests

```bash
./scripts/validate.sh
```

---

## Adding a New Environment

To deploy to a second cluster (e.g., `production`):

```bash
# 1. Create the cluster bootstrap
cp -r clusters/staging clusters/production
# Update gotk-sync.yaml path → ./clusters/production

# 2. Create the env overlay
cp -r apps/staging apps/production
# Add kustomize patches (replicas, resources, env-specific config)

# 3. Update ArtifactGenerator to include the new env
# clusters/production/artifacts.yaml → add apps/production/**

# 4. Bootstrap Flux on the new cluster
flux bootstrap github \
  --owner=traipoap \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/production \
  --personal
```

Each cluster has its own `flux-system/` + Kustomizations. Shared manifests live in `apps/base/`.

---

## Maintenance Checklist

- [ ] Pin Helm chart versions (currently using `>=` ranges)
- [ ] Switch from self-signed CA to Let's Encrypt for production domains
- [ ] Add PodDisruptionBudgets for stateful components (Quickwit, Vector)
- [ ] Add NetworkPolicies for namespace isolation
- [ ] Configure Flux alerts (Slack/Email) via `Alert` resources
- [ ] Add Kyverno/OPA policies for security enforcement
- [ ] Multi-environment promotion (staging → production)
- [ ] Add Velero backup for etcd + PVCs
- [ ] Add image vulnerability scanning in CI pipeline
