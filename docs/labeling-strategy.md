# BSIS Kubernetes Labeling Strategy

This document defines a consistent Kubernetes labeling standard for BSIS workloads.

## Goals

- Make workload ownership and purpose clear.
- Keep selectors stable and predictable.
- Improve filtering in `kubectl`, Argo CD, and observability tools.

## Required labels

Apply these labels to every workload resource (`Deployment`, `Service`, `HPA`, `Ingress`, `PDB`, `ServiceAccount`, `NetworkPolicy`).

- `app.kubernetes.io/name`: workload name (for example `identity-service`).
- `app.kubernetes.io/component`: logical component (`identity`, `api`, `frontend`).
- `app.kubernetes.io/part-of`: platform name (`bsis`).
- `app.kubernetes.io/managed-by`: deployment manager (`argocd`).
- `app.kubernetes.io/environment`: deployment environment (`dev`, `stage`, `prod`).
- `app.kubernetes.io/instance`: unique workload instance in an environment (for example `identity-service-dev`).

## BSIS service mapping

Use the following canonical names:

| Service | `app.kubernetes.io/name` | `app.kubernetes.io/component` |
|---|---|---|
| Identity Service | `identity-service` | `identity` |
| Core API | `core-api` | `api` |
| Main BSIS Frontend | `main-bsis-frontend` | `frontend` |
| Facility Frontend | `facility-frontend` | `frontend` |

## Naming conventions

Use lowercase kebab-case for all names. Avoid underscores, spaces, and uppercase letters.

- Service slug format: `<service-name>` (for example `core-api`).
- Environment format: `dev`, `stage`, `prod`.
- Instance format: `<service-name>-<env>` (for example `core-api-prod`).

### Kubernetes resource names

- Namespace: `app-<env>` (for example `app-dev`).
- Deployment/Service/HPA/Ingress/PDB/ServiceAccount base name: `<service-name>`.
- Secret consumed by workload: `<service-name>-secrets`.
- NetworkPolicy name: `<service-name>-default-deny-ingress`.

### Argo CD and path naming

- Argo CD application name: `<service-name>-<env>`.
- Overlay path: `apps/<service-name>/overlays/<env>`.
- Base path: `apps/<service-name>/base`.

### DNS and TLS naming

- Host format: `<service-name>-<env>.example.com` for non-prod.
- Prod host format: `<service-name>.example.com`.
- TLS secret format: `<service-name>-<env>-tls` (prod can be `<service-name>-prod-tls` for consistency).

### Example mapping

| Service | dev instance | stage instance | prod instance |
|---|---|---|---|
| Identity Service | `identity-service-dev` | `identity-service-stage` | `identity-service-prod` |
| Core API | `core-api-dev` | `core-api-stage` | `core-api-prod` |
| Main BSIS Frontend | `main-bsis-frontend-dev` | `main-bsis-frontend-stage` | `main-bsis-frontend-prod` |
| Facility Frontend | `facility-frontend-dev` | `facility-frontend-stage` | `facility-frontend-prod` |

## Selector guidance

- Use `app.kubernetes.io/name` as the primary selector key.
- Do not use environment labels in `Service` selectors unless required.
- Keep selector labels immutable after first deployment.

## Example

```yaml
metadata:
  labels:
    app.kubernetes.io/name: core-api
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: bsis
    app.kubernetes.io/managed-by: argocd
    app.kubernetes.io/environment: prod
    app.kubernetes.io/instance: core-api-prod
```
