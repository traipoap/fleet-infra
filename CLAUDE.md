CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Project Overview

This is a GitOps and Kubernetes infrastructure repository. It uses Kustomize for manifest management and Flux CD for continuous delivery and cluster state synchronization. The repository defines the foundational infrastructure, security, observability, and core application deployments across different environments (e.ability to scale with more clusters).

Architecture

The repository is organized into three primary layers:

1. Applications (/applications):
  - Contains Kustomize overlays for deploying the application stack.
  - The base directory defines a standard deployment including a frontend, backend, and an API gateway (using Kubernetes Gateway API).
2. Infrastructure (/infrastructure):
  - Defines shared platform services organized by functional area:
      - Logging: Managed via Quickwit for indexing and Vector for log routing/collection.
    - Networking: Includes NFS provisioning for persistent storage.
    - Observability: Provides the monitoring stack, including Prometheus, Grafana, and Kiali (for Istio/Service Mesh visibility).
    - Security: Handles certificate management via cert-manager.
3. Clusters (/clusters):
  - Contains Flux CD configurations for specific environments.
  - The dev cluster configuration manages the synchronization of infrastructure and applications defined in the repo.

Development & Inspection

Since this is a declarative infrastructure repository, "development" primarily involves modifying manifests and verifying the resulting Kubernetes objects.

Inspecting Manifests

To see the fully rendered Kubernetes manifests for any component or application, use kustomize build. This is essential to verify how overlays (like patches) affect the base configuration.

# Inspect the base application stack
kustomize build applications/base

# Inspect a specific infrastructure component (e.g., Quickwit indexer)
kustomize build infrastructure/logging/quickwit/quickwit-indexer

Workflow Note

Changes to this repository are intended to be reconciled by Flux CD in the target clusters. Always ensure that kustomization.yaml files correctly reference all necessary resources and that the paths used in kustomize build reflect the intended state.
