# Architecture & Phased Delivery Plan

This document captures the design of the self-hosted CI/CD platform built in this repo and the phased plan for getting it there.

## Goals

1. **Real CI/CD, not pipeline YAML theater.** Every piece (runners, build, registry, deploy, autoscale, cache, observability) is owned and operated, not consumed as a managed service.
2. **Production-grade defaults.** RBAC-scoped service accounts, rootless container builds, no privileged pods, ephemeral runners (one job per pod, then destroyed), zero-downtime deploys.
3. **Observable end-to-end.** Build duration, queue depth, runner pool size, deploy lead time, and API SLOs (latency / error rate / RPS) all visible in Grafana.
4. **Reproducible.** Anyone can `git clone` this repo and a sibling `Fifa_API` clone and stand the whole stack up locally on Kubernetes in under 30 minutes.

## Components

### Cluster
- **Local**: [k3d](https://k3d.io/) v5.8.3, k3s v1.33.6, 1 server + 3 agents, loadbalancer mapping `localhost:8080→80` and `localhost:8443→443`. Cluster name `fifa-cicd`.
- **Future cloud target**: any of EKS / GKE / AKS — the manifests and Helm values stay portable; only the StorageClass / LoadBalancer / DNS bits differ.

### CI/CD runtime
- **Actions Runner Controller (ARC)** — modern scale-set flavor (`AutoscalingRunnerSet` CRD).
- Authenticates to GitHub via a **GitHub App** (preferred over PAT — finer-grained permissions, no human-tied secrets).
- Runners are **ephemeral**: one Job → one runner pod → destroyed.
- Scales 0→N based on GitHub job queue depth (native to `AutoscalingRunnerSet`).

### Container build
- **Kaniko** for per-job rootless builds (TBD: may swap to BuildKit rootless once registry-mode caching is wired).
- No DinD, no `--privileged`. Build runs as a non-root container; pushes to the registry using a credential mounted from a Kubernetes Secret.

### Registry
- TBD: GHCR (free, integrates with the GitHub App) vs in-cluster Harbor / Zot. Default plan is GHCR for simplicity; Harbor is a stretch goal if we want a "fully in-cluster" demo.

### Application
- [`ajugulum123/Fifa_API`](https://github.com/ajugulum123/Fifa_API) — Apollo Server 4 GraphQL API on Node.js 18 + TypeScript 5, serves 5,682 FIFA players from a CSV, JWT auth + RBAC. See its README for details.
- **TLS termination decision (open)**: option (a) strip in-app TLS in production, terminate at the ingress (Traefik + cert-manager) — recommended; option (b) keep in-app TLS and pass-through at ingress.

### Deployment style
- TBD: straight `kubectl apply` from the workflow vs Helm vs Argo CD (pull-based GitOps). Default plan is Helm + Argo CD as a stretch.

### Networking & ingress
- **Traefik** (k3s default ingress controller, already running)
- **cert-manager** for TLS certificates (Let's Encrypt staging in non-prod, real once a public DNS name is in play; for k3d we'll use self-signed via cert-manager for the demo flow)
- NetworkPolicies to constrain pod-to-pod traffic (runners can't reach the API namespace except through the deploy SA's RBAC scope)

### Caching
- **MinIO** in-cluster as the S3-compatible backing store
- An Actions cache action that speaks the S3 protocol (e.g. `tespkg/actions-cache`) so self-hosted runners get the GitHub Actions cache experience without GitHub's hosted cache service
- BuildKit / Kaniko **registry-mode cache** for image layers
- A PVC mounted into runners for npm / Go module caches across jobs

### Secrets
- All credentials (GitHub App private key, registry creds, JWT secrets, admin password) live in Kubernetes Secrets, never in Git, never in env files committed to the repo
- The application's `.env`-style config moves to a ConfigMap (non-sensitive) + Secret (sensitive), wired in via `envFrom`
- Optional stretch: external-secrets-operator + a backing store (Vault / AWS Secrets Manager / SOPS-encrypted files in this repo)

### Observability
- **kube-prometheus-stack** Helm chart: Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics
- ServiceMonitors for the ARC controller, runner pods, and the API itself
- API instrumented with `prometheus-fastapi-instrumentator`-equivalent middleware for Apollo / Express
- Grafana dashboards: API SLOs (latency / error rate / RPS), runner pool utilization, build & deploy timings
- Stretch: Loki for logs, Tempo for traces

## Phased delivery plan

| Phase | Deliverable | Repo(s) touched |
|---|---|---|
| **1 — Local cluster** ✅ | k3d cluster up, kubectl + helm working | (infra-only) |
| **2a — Containerize** | Dockerfile in `Fifa_API`, builds locally, runs end-to-end | `Fifa_API` |
| **2b — Tests** | Vitest test suite covering auth / RBAC / depth / rate / 1 query / 1 mutation | `Fifa_API` |
| **2c — K8s manifests** | Namespace, Deployment, Service, ConfigMap, Secret, Ingress for the API | this repo |
| **3 — GitHub App + ARC** | App created, secret-mounted, ARC installed, smoke runner picks up a job | this repo |
| **4 — CI workflow** | `Fifa_API` workflow runs Vitest in an ARC runner, gates merge | `Fifa_API` |
| **5 — Build & push** | Kaniko builds image, pushes to registry on `push: main` | `Fifa_API` |
| **6 — CD workflow** | This repo applies new image SHA, waits for rollout health, fails loudly | this repo |
| **7 — Autoscaling proven** | Queue 10 jobs, watch the runner pool scale 0→N→0; record video | this repo |
| **8 — Shared cache** | MinIO in-cluster + Actions cache action + BuildKit registry cache | this repo + `Fifa_API` |
| **9 — Observability** | kube-prometheus-stack, ServiceMonitors, Grafana dashboards | this repo |
| **10 — Polish** | Architecture diagram, runbook, demo recording, README polish | this repo |

## Open architectural decisions

These need to be locked before the implementation phases that depend on them; some can be deferred a bit.

| # | Decision | Default plan | Block to |
|---|---|---|---|
| 1 | TLS termination — at ingress (a) or in-app (b) | (a) at ingress | Phase 2c |
| 2 | Image registry — GHCR or in-cluster Harbor | GHCR for simplicity, Harbor as stretch | Phase 5 |
| 3 | Build tool — Kaniko per-job or BuildKit rootless | Kaniko first, swap to BuildKit if cache becomes a bottleneck | Phase 5 |
| 4 | Deploy style — `kubectl apply` / Helm / Argo CD | Helm now, Argo CD as Phase 10 stretch | Phase 6 |
| 5 | Test framework — Vitest / Jest / Mocha | Vitest | Phase 2b |
| 6 | Cloud target — local-only or also EKS/GKE/AKS | Local-only for v1, cloud-readiness as design constraint | Phase 10 |

## Repo layout (planned)

```
KubernetesDeploymentFifaAPI/
├── README.md
├── LICENSE
├── .gitignore
├── docs/
│   ├── ARCHITECTURE.md          this file
│   ├── RUNBOOK.md               (Phase 10) day-2 ops procedures
│   └── DEMO.md                  (Phase 10) how to reproduce the demo end-to-end
├── clusters/
│   └── local-k3d/
│       └── README.md            how to (re)create the cluster — the exact k3d command
├── manifests/                   raw K8s manifests, organized by component
│   ├── namespaces/
│   ├── fifa-api/                Deployment, Service, ConfigMap, Secret, Ingress
│   ├── arc/                     ARC system + runner scale set
│   ├── cert-manager/
│   ├── observability/           kube-prometheus-stack values + ServiceMonitors
│   ├── cache/                   MinIO + cache server
│   └── rbac/                    ServiceAccounts, Roles, RoleBindings per workload
├── charts/                      Helm charts (one per non-trivial component if we go Helm-heavy)
└── .github/
    └── workflows/               this repo's own CI (lint manifests, kubeconform, helm lint, etc.)
                                 plus the CD workflow that watches Fifa_API for new images
```

## Drift control

When a pattern is reused 3+ times across phases, promote it to a dedicated rule under `.cursor/rules/` (per the workspace's `self-improvement.mdc`).
