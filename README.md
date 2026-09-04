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
│       │   ├── kustomization.yaml   # Bundles components + sync
│       │   ├── gotk-components.yaml # Flux CRDs + controllers
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
│       │   │   ├── kustomization.yaml
│       │   │   ├── vector-source.yaml   # HelmRepository (helm.vector.dev)
│       │   │   └── vector-release.yaml  # HelmRelease (Agent, syslog+k8s → Quickwit)
│       │   └── quickwit/            # Quickwit (log store)
│       │       ├── kustomization.yaml
│       │       ├── quickwit-source.yaml  # HelmRepository (helm.quickwit.io) + Namespace
│       │       ├── quickwit-release.yaml # HelmRelease (valuesFrom: S3 secret)
│       │       ├── quickwit-indexer.yaml
│       │       └── job-create-index.yaml
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
│   └── validate.sh                  # flux-schema validation (YAML + kustomize + helm)
│
├── .gitignore
└── README.md
```

---

## Flux CD Resources

### Sources

| Kind | Name | Namespace | Purpose |
|------|------|-----------|---------|
| `GitRepository` | `flux-system` | `flux-system` | Points to this repo (main branch) |
| `ArtifactGenerator` | `flux-system` | `flux-system` | Packages `infrastructure/**` and `apps/**` into artifacts |
| `HelmRepository` | `jetstack` | `cert-manager` | cert-manager OCI charts (`quay.io/jetstack/charts`) |
| `HelmRepository` | `external-secrets` | `external-secrets` | External Secrets charts (`charts.external-secrets.io`) |
| `HelmRepository` | `vector-repo` | `lumina` | Vector charts (`helm.vector.dev`) |
| `HelmRepository` | `quickwit-repo` | `lumina` | Quickwit charts (`helm.quickwit.io`) |
| `HelmRepository` | `nfs-subdir-external-provisioner` | `nfs` | NFS provisioner (`kubernetes-sigs.github.io`) |
| `HelmRepository` | `ww-gitops` | `flux-system` | Weave GitOps OCI charts (`ghcr.io/weaveworks/charts`) |

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

| Name | Namespace | Chart | Source | Notes |
|------|-----------|-------|--------|-------|
| `cert-manager` | `cert-manager` | `cert-manager` (jetstack OCI) | `jetstack` | `installCRDs: true` |
| `external-secrets` | `external-secrets` | `external-secrets` | `external-secrets` | `crds: Create` |
| `vector` | `lumina` | `vector` | `vector-repo` (helm.vector.dev) | Agent mode, syslog+k8s → Quickwit |
| `quickwit` | `lumina` | `quickwit` | `quickwit-repo` (helm.quickwit.io) | `valuesFrom: quickwit-s3-secret-values` |
| `nfs-subdir-external-provisioner` | `nfs` | `nfs-subdir-external-provisioner` | `nfs-subdir-external-provisioner` | `valuesFrom: nfs-provisioner-secret-values` |
| `ww-gitops` | `flux-system` | `weave-gitops` | `ww-gitops` | `valuesFrom: weave-secret-values` |

### Image Automation (ImageUpdateAutomation)

| Name | Image | Policy | Update Path | Strategy |
|------|-------|--------|-------------|----------|
| `backend` | `ghcr.io/traipoap/backend` | semver `>=0.0.0` | `./apps/staging` | Setters |
| `frontend` | `ghcr.io/traipoap/frontend` | semver `>=0.0.0` | `./apps/staging` | Setters |

> Flux auto-updates image tags in `apps/staging/` and commits as `fluxcdbot`.

### External Secrets

| Kind | Name | AWS SM Key | K8s Secret | Purpose |
|------|------|-----------|------------|---------|
| `ClusterSecretStore` | `secretstore-aws` | region: `ap-southeast-2` | — | Provider config (auth via `awssm-secret`) |
| `ExternalSecret` | `backend-jwt-secret` | `dev/backend/jwt` → `JWT_SECRET` | `jwt-secret` | JWT for backend |
| `ExternalSecret` | `github-registry-secret` | `dev/github-registry/frontend-backend` | `github-registry` (dockerconfigjson) | GHCR image pull creds |
| `ExternalSecret` | `nfs-provisioner-values` | `dev/nfs/config` → `SERVER`, `PATH` | `nfs-provisioner-secret-values` (values.yaml) | NFS server endpoint |
| `ExternalSecret` | `quickwit-s3-values` | `dev/lumina/quickwit/s3` → `access_key_id`, `secret_access_key`, `endpoint` | `quickwit-s3-secret-values` (values.yaml) | Quickwit S3/Garage storage |
| `ExternalSecret` | `weave-values` | `dev/flux-system/weave` → `username`, `passwordHash` | `weave-secret-values` (values.yaml) | Weave GitOps admin |

### Security & Cost Optimization

| Kind | Name | Scope | Purpose |
|------|------|-------|---------|
| `LimitRange` | `cpu-defaults` | All 7 namespaces | Default CPU requests/limits per container |
| `ClusterPolicy` (Kyverno) | `set-default-resource-requests` | Cluster-wide | Mutate Pods: inject `resources.requests/limits` if missing |

**LimitRange defaults:**

| Namespace | Default Request | Default Limit |
|-----------|----------------|---------------|
| `istio-system` | 50m | 500m |
| All others (`cert-manager`, `external-secrets`, `flux-system`, `kube-system`, `lumina`, `nfs`) | 10m | 100m |

**Kyverno mutation (applies to all Pods without explicit resource specs):**
```
requests: cpu=10m, memory=1Mi
limits:   cpu=200m, memory=450Mi
```

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

- **JWT_SECRET**: AWS SM (`dev/backend/jwt`) → K8s Secret `jwt-secret` → env var on backend
- **Registry credentials**: AWS SM (`dev/github-registry/frontend-backend`) → K8s Secret `github-registry` (dockerconfigjson template) → `imagePullSecrets`
- **Weave GitOps admin**: K8s Secret `weave-secret-values` → HelmRelease `valuesFrom` (no hardcoded passwords)

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
ImageRepository (poll every 5m, namespace: lumina)
    │
    ▼
ImagePolicy (semver: pick highest ≥ 0.0.0)
    │
    ▼
ImageUpdateAutomation (Setters strategy)
    │  └─ updates kustomize `images` field in apps/staging/kustomization.yaml
    │     commit as fluxcdbot → push to main
    │
    ▼
Flux Kustomization: apps (detects diff → deploys new image)
```

**Setters mechanism:** Flux updates the `newTag` in `apps/staging/kustomization.yaml`:
```yaml
images:
  - name: ghcr.io/traipoap/backend
    newTag: "0.0.47"  # ← Flux auto-updates this value
  - name: ghcr.io/traipoap/frontend
    newTag: "0.0.28"  # ← Flux auto-updates this value
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

Uses **flux-schema** (Flux Schema plugin) with the [Flux Ecosystem Catalog](https://schemas.fluxoperator.dev/):

```bash
# Validate all YAML + kustomize overlays
./scripts/validate.sh

# Include Helm chart rendering
./scripts/validate.sh -H

# Custom directory + output bundle
./scripts/validate.sh -d ./infrastructure -b /tmp/bundle.yaml

# Exclude specific dirs
./scripts/validate.sh -e .scannerwork
```

**Validates:** Standalone YAML, Kustomize overlays, Helm charts (optional `-H`)
**Prerequisites:** `flux-schema` ≥ 0.9, `kustomize`/`kubectl`, `helm` ≥ 4.0 (for `-H`)

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

- [x] Pin Helm chart versions (currently using `>=` ranges)
- [ ] Switch from self-signed CA to Let's Encrypt for production domains
- [ ] Add PodDisruptionBudgets for stateful components (Quickwit, Vector)
- [ ] Add NetworkPolicies for namespace isolation
- [ ] Configure Flux alerts (Slack/Email) via `Alert` resources
- [ ] Add Kyverno/OPA policies for security enforcement
- [ ] Multi-environment promotion (staging → production)
- [ ] Add Velero backup for etcd + PVCs
- [x] Add image vulnerability scanning in CI pipeline
