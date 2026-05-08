# CivicGrid GitOps

Environment configuration for CivicGrid services. Watched by ArgoCD running in the same cluster.

## Layout

apps/
  civicgrid-api/
    application.yaml   - ArgoCD Application manifest
    values.yaml        - Helm values overrides

## How it works

ArgoCD watches this repo and the civicgrid-api repo. When values.yaml changes, ArgoCD reconciles the cluster automatically. Auto-sync, prune, and self-heal are all enabled.

## Adding new environments

Each environment gets its own folder under apps/ (e.g. apps/civicgrid-api-staging/). Update the namespace in application.yaml and values appropriately, then apply the new manifest into the argocd namespace.

## Secrets

Secrets are NOT managed via this repo. They are created in-cluster via kubectl create secret (manually) for now. Sealed Secrets or External Secrets Operator will replace this in a follow-up.
