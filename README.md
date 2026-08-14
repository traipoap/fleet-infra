## Repository Structure
```.
├── applications
│   └── base
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml
│       ├── cert-lumina.yaml
│       ├── configmap.yaml
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml
│       ├── gateway.yaml
│       ├── httproute.yaml
│       ├── ingress.yaml
│       ├── kustomization.yaml
│       └── namespace.yaml
├── clusters
│   └── dev
│       └── flux-system
│           ├── gotk-components.yaml
│           ├── gotk-sync.yaml
│           └── kustomization.yaml
├── infrastructure
│   ├── kustomization.yaml
│   ├── logging
│   │   ├── kustomization.yaml
│   │   ├── otel-config.yaml
│   │   ├── quickwit
│   │   │   ├── job-create-index.yaml
│   │   │   ├── kustomization.yaml
│   │   │   ├── quickwit-indexer.yaml
│   │   │   ├── quickwit-release.yaml
│   │   │   └── temp.yaml
│   │   └── vector
│   │       ├── kustomization.yaml
│   │       ├── values.yaml
│   │       └── vector-release.yaml
│   ├── networking
│   │   ├── kustomization.yaml
│   │   └── nfs
│   │       ├── kustomization.yaml
│   │       └── nfs-subdir-external-provisioner-release.yaml
│   ├── observability
│   │   ├── grafana
│   │   │   └── kustomization.yaml
│   │   ├── kialia
│   │   │   └── kustomization.yaml
│   │   ├── kustomization.yaml
│   │   └── prometheus
│   │       └── kustomization.yaml
│   └── security
│       ├── cert-manager
│       │   ├── cert-manager-config.yaml
│       │   └── kustomization.yaml
│       └── kustomization.yaml
├── README.md
└── sources
    ├── git-repo.yaml
    ├── kustomization.yaml
    ├── nfs-repo.yaml
    ├── quickwit-repo.yaml
    ├── vector-repo.yaml
    └── weave-gitops-dashboard.yaml

19 directories, 42 files
```
