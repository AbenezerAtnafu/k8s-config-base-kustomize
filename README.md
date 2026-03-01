# BSIS CCE Kubernetes Baseline

Baseline deployment template for Huawei CCE using Kustomize overlays and Argo CD GitOps.

## Structure

- `apps/base`: reusable stateless workload manifests
- `apps/overlays/development|staging|production`: environment overrides
- `platform/argocd`: Argo CD project and applications
- `.github/workflows`: image build/push and promotion workflows
- `docs`: operational and security guidance

## Prerequisites

- CCE cluster and Argo CD installed in namespace `argocd`
- GHCR package path available (`ghcr.io/habtec/bsis-app`)
- GitHub Actions permissions include `packages: write`

## Bootstrap

1. Confirm repo URL in `platform/argocd/*.yaml` points to your GitHub repo.
2. Confirm GHCR path (`ghcr.io/habtec/bsis-app`) in overlays/workflows/manifests.
3. Set your ACME email in `platform/cert-manager-config/clusterissuer-letsencrypt-production.yaml`.
4. If needed, set CCE ELB annotations in `platform/argocd/app-ingress-nginx.yaml`.
5. Create namespace secrets:
   - See `docs/secrets-and-rbac.md`.
6. Apply Argo CD resources:

```bash
kubectl apply -k platform/argocd
```

## Local validation

```bash
kustomize build apps/overlays/development
kustomize build apps/overlays/staging
kustomize build apps/overlays/production
```

## Deployment flow

1. Push to `main` -> GitHub Action builds and pushes image to GHCR.
2. Workflow updates `apps/overlays/development/kustomization.yaml` with immutable image tag.
3. Argo CD auto-syncs development.
4. Promote to staging/production via `Promote Image Tag` workflow.

## References

- `docs/secrets-and-rbac.md`
- `docs/secrets-config-strategy.md`
- `docs/vault-in-cluster-bootstrap.md`
- `docs/vault-rotation-runbook.md`
- `docs/operations-baseline.md`
- `docs/observability-retention-policy.md`
