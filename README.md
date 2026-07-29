# CivicGrid GitOps

Declarative cluster configuration for **CivicGrid**, a B2B API serving verified civic data for 3,064 U.S. cities.

ArgoCD runs inside the `civicgrid-prod` DigitalOcean Kubernetes cluster and reconciles everything in this repo continuously. Every Application here has `prune` and `selfHeal` enabled — manual `kubectl edit` against these resources is reverted automatically.

Related repositories:

| Repo | Contains |
|---|---|
| [`civicgrid-api`](https://github.com/Kwamib/civicgrid-api) | FastAPI application code and the `charts/civicgrid-api` Helm chart |
| `civicgrid-infra` | Terraform (Terraform Cloud) — cluster, DNS, and the in-cluster Secrets consumed below |
| **this repo** | ArgoCD Applications, platform components, and environment values |

## Layout

```
apps/
  civicgrid-api/
    application.yaml          ArgoCD Application (multi-source)
    values.yaml               Helm values for this environment

infra/
  cert-manager/               cert-manager v1.16.2 (Jetstack chart)
  cert-manager-issuers/       ClusterIssuer for Gateway certificates
  envoy-gateway/              Envoy Gateway v1.5.4 (Gateway API controller)
  gateway/                    GatewayClass "eg"
  civicgrid-gateway/          Gateway + HTTPRoute for api.civicgrid.org
  metrics-server/             metrics-server v3.12.2 (HPA metrics source)
  observability/              kube-prometheus-stack v65.5.1

grafana-dashboards/
  civicgrid-api-live.json     Golden-signals dashboard for the API
```

## How the API Application works

`apps/civicgrid-api/application.yaml` uses ArgoCD's **multi-source** pattern to keep the chart and its environment values in separate repos:

- **Source 1** — the Helm chart, from `civicgrid-api` at `charts/civicgrid-api`
- **Source 2** — this repo, referenced as `$values`, supplying `apps/civicgrid-api/values.yaml`

The chart ships with the application it deploys; the values that make it *this* environment stay here. Adding a second environment means a new folder under `apps/` with its own `values.yaml`, not a fork of the chart.

Sync options: `CreateNamespace=true` and `ServerSideApply=true`, deploying into the `civicgrid` namespace.

## Sync ordering

Applications are ordered with `argocd.argoproj.io/sync-wave` so the platform is ready before the workload lands:

| Wave | Applications |
|---|---|
| 0 | cert-manager, envoy-gateway, metrics-server, GatewayClass |
| 1 | cert-manager ClusterIssuer, civicgrid Gateway + HTTPRoute |
| 2 | kube-prometheus-stack |
| 3 | civicgrid-api |

## Ingress and TLS

Traffic reaches the API through the Gateway API rather than an Ingress controller:

```
api.civicgrid.org → Gateway (envoy) :443 → HTTPRoute → Service civicgrid-civicgrid-api:8000
```

The Gateway terminates TLS using a certificate issued by cert-manager into the `civicgrid-gateway-tls` Secret. The current ClusterIssuer is `selfsigned-issuer` — the public certificate is presented by Cloudflare at the edge, and this covers the origin leg only.

## Autoscaling and metrics

The API runs with an HPA (1–4 replicas, 80% CPU target), backed by metrics-server. A `ServiceMonitor` is enabled at a 30s interval, so Prometheus scrapes the API automatically and the Grafana dashboard in `grafana-dashboards/` renders against it.

## Secrets

No secret material is stored in this repo. Values reference Secrets by name only:

| Secret | Key | Used for |
|---|---|---|
| `civicgrid-database` | `DATABASE_URL` | Supabase Postgres connection |
| `civicgrid-admin` | `ADMIN_TOKEN` | Admin endpoint authentication |
| `civicgrid-admin` | `SUPABASE_JWT_SECRET` | JWT verification on `/me` endpoints |

These are created in-cluster by Terraform in `civicgrid-infra`. Sealed Secrets is a planned follow-up so that secret *definitions* can live in Git alongside everything else.

## Known gaps

Kept here deliberately rather than in a private tracker:

- **Applications are applied individually.** There is no root app-of-apps Application; each manifest under `apps/` and `infra/` is applied into the `argocd` namespace once, after which ArgoCD self-manages it.
- **The API image tag is `latest`** with `pullPolicy: Always`. CI does not currently commit a resolved digest here, so a deploy is not reproducible from this repo alone. Pinning the tag from CI is the next meaningful change.
- **Redis is disabled.** The Bitnami chart URL stopped resolving after their August 2025 catalog restructure, so the rate limiter falls back to in-memory. That is correct at one replica but drifts if the HPA scales out.
