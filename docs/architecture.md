# Architecture

## Overview

| Component | Purpose |
|---|---|
| **web** | Static frontend served by `nginx-unprivileged`; calls the API under `/api` |
| **api** | FastAPI service: task CRUD, `/healthz`, `/readyz`, `/metrics`, `/api/version` |
| **postgres** | Persistent store (StatefulSet + PVC on k3s `local-path`) |
| **redis** | Cache for read-heavy endpoints |

## Environments

One k3s cluster, two namespaces:

| | staging | prod |
|---|---|---|
| Namespace | `taskboard-staging` | `taskboard-prod` |
| Deployed by | every merge to `main` | release tag `v*` + manual approval |
| Hostname | `staging.lab.<domain>` | `app.lab.<domain>` |
| Replicas (HPA) | 1–3 | 2–6 |

## Delivery flow

1. A pull request runs CI on GitHub-hosted runners:
   - lint and unit tests
   - Dockerfile lint
   - image and config scans
   - Helm lint and schema validation
   - a throwaway **kind** cluster where the chart is installed and smoke-tested
2. A merge to `main` builds images, pushes them to GHCR tagged `sha-<commit>` (with SBOM and provenance attached), and signs them with cosign.
3. `deploy-staging` runs on an **ARC runner pod inside k3s**. It uses a namespace-scoped ServiceAccount, so no kubeconfig ever leaves the cluster. It runs `helm upgrade --install --atomic --wait`, then a smoke test.
4. A `v*` tag waits for approval on the `production` environment, then deploys to prod the same way.
5. `rollback.yml` can roll back to any previous Helm revision.

## Traffic flow

```
Client ─► DNS *.lab.<domain> ─► k3s ServiceLB ─► NGINX Gateway Fabric (TLS, wildcard cert)
       ─► HTTPRoute ─► web / api Services ─► Pods
```

TLS certificates come from Let's Encrypt via cert-manager. Validation uses DNS-01 against Cloudflare, so the lab never needs to be reachable from the internet.

## Zero-downtime rollouts

A rolling update setting alone does not guarantee zero downtime. This chart combines:

- `maxSurge: 1`, `maxUnavailable: 0`, `minReadySeconds`
- readiness probes that check PostgreSQL and Redis
- a `preStop` delay and graceful shutdown, so in-flight requests finish
- at least 2 replicas, a PDB, and pods spread across both nodes
- backward-compatible database migrations, run as a Helm pre-upgrade hook

The claim is verified by load-testing during a rollout (see `docs/evidence/`).

## Observability

- Metrics from **kube-prometheus-stack**: node-exporter, kube-state-metrics, the API's `/metrics`, NGINX Gateway Fabric metrics, and blackbox probes of the public URLs
- **Grafana** dashboards stored as JSON in `platform/monitoring/dashboards/`
- **Alertmanager** sends alerts to Telegram; every rule links to a runbook in `docs/runbooks.md`
