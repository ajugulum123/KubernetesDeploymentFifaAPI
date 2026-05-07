# Architecture and Phased Delivery Plan

This document captures the design of the self-hosted CI/CD platform built in this repo and the phased plan for getting it there.

## Goals

1. **Real CI/CD, not pipeline YAML theater.** Every piece (runners, build, registry, deploy, autoscale, cache, observability) is owned and operated, not consumed as a managed service.
2. **Production-grade defaults.** RBAC-scoped service accounts, rootless container builds, no privileged pods, ephemeral runners (one job per pod, then destroyed), zero-downtime deploys.
3. **Observable end-to-end.** Build duration, queue depth, runner pool size, deploy lead time and API SLOs (latency, error rate, RPS) all visible in Grafana.
4. **Reproducible.** Anyone can `git clone` this repo and a sibling `Fifa_API` clone and stand the whole stack up locally on Kubernetes in under 30 minutes.

## Components

### Cluster
- **Local**: [k3d](https://k3d.io/) v5.8.3, k3s v1.33.6, 1 server + 3 agents, loadbalancer mapping `localhost:8080` to `:80` and `localhost:8443` to `:443`. Cluster name `fifa-cicd`.
- **Future cloud target**: any of EKS, GKE or AKS. The manifests and Helm values stay portable. Only the StorageClass, LoadBalancer and DNS bits differ.

### CI/CD runtime
- **Actions Runner Controller (ARC)** in the modern scale-set flavor (`AutoscalingRunnerSet` CRD).
- Authenticates to GitHub via a **GitHub App** (preferred over PAT for finer-grained permissions and no human-tied secrets).
- Runners are **ephemeral**: one Job leads to one runner pod which is then destroyed.
- Scales 0 to N based on GitHub job queue depth (native to `AutoscalingRunnerSet`).

### Container build
- **Rootless BuildKit** running in-cluster as a Deployment. Workflows talk to it via `buildctl`. No DinD, no `--privileged`, no per-job daemon.
- Pushes to GHCR using a credential mounted from a Kubernetes Secret.

### Registry
- **GHCR** (GitHub Container Registry). Free, integrates cleanly with the GitHub App, no separate auth flow required for in-org images. In-cluster Harbor or Zot remain as a stretch goal if a fully-in-cluster demo is desired.

### Application
- [`ajugulum123/Fifa_API`](https://github.com/ajugulum123/Fifa_API). Apollo Server 4 GraphQL API on Node.js 18 with TypeScript 5. Serves 5,682 FIFA players from a CSV. JWT auth plus RBAC. See its README for details.
- **TLS termination**: terminated at the Ingress (Traefik + cert-manager) in the cluster. The app exposes plain HTTP on its container port when run inside Kubernetes (toggled via `DISABLE_INLINE_TLS=true`). Local dev still uses in-app TLS via the existing `gen:certs` script.

### Deployment style
- **Plain Kubernetes manifests under `kubernetes/`**, applied by the CD workflow with `kubectl apply -k` (Kustomize for env overlays). Helm is reserved for third-party charts (ARC, kube-prometheus-stack, MinIO). Argo CD is documented as a Phase 17 stretch goal.

### Networking and ingress
- **Traefik** (k3s default ingress controller, already running)
- **cert-manager** for TLS certificates. Let's Encrypt staging in non-prod once a public DNS name is in play. For k3d the demo uses self-signed via cert-manager.
- NetworkPolicies to constrain pod-to-pod traffic. Runners cannot reach the API namespace except through the deploy ServiceAccount's RBAC scope.

### Caching
- **MinIO** in-cluster as the S3-compatible backing store
- An Actions cache action that speaks the S3 protocol (e.g. `tespkg/actions-cache`) so self-hosted runners get the GitHub Actions cache experience without GitHub's hosted cache service
- BuildKit **registry-mode cache** for image layers, pointed at GHCR
- A PVC mounted into runners for npm and Go module caches across jobs

### Secrets
- All credentials (GitHub App private key, registry creds, JWT secrets, admin password) live in Kubernetes Secrets, never in Git, never in env files committed to the repo
- The application's `.env`-style config moves to a ConfigMap (non-sensitive) plus Secret (sensitive), wired in via `envFrom`
- Optional stretch: external-secrets-operator with a backing store (Vault, AWS Secrets Manager or SOPS-encrypted files in this repo)

### Observability
- **kube-prometheus-stack** Helm chart: Prometheus, Grafana, Alertmanager, node-exporter and kube-state-metrics
- ServiceMonitors for the ARC controller, runner pods and the API itself
- API instrumented with `prom-client` plus `express-prom-bundle` for request count, latency histograms and error rates
- Grafana dashboards covering API SLOs (latency, error rate, RPS), runner pool utilization, build and deploy timings
- Stretch: Loki for logs, Tempo for traces

## Phased delivery plan

| Phase | Deliverable | Repo(s) touched |
|---|---|---|
| **1. Local cluster** (done) | k3d cluster up, kubectl and helm working | (infra-only) |
| **2. GitHub App and ARC** (in progress) | App created, ARC controller installed, runner scale set deployed, runner picks up a job | this repo |
| **3. Containerize** | multi-stage Dockerfile in `Fifa_API`, builds locally, runs end-to-end | `Fifa_API` |
| **4. Tests** | Vitest test suite covering auth, RBAC, depth, rate plus one query and one mutation | `Fifa_API` |
| **5. Custom runner image** | Runner image with kubectl, Docker CLI, curl, jq, Node 20. Pushed to GHCR. Scale set uses it. | this repo |
| **6. K8s manifests** | Namespace, Deployment, Service, ConfigMap, Secret, Ingress for the API | this repo |
| **7. CI workflow** | `Fifa_API` workflow runs Vitest in an ARC runner, gates merge | `Fifa_API` |
| **8. Build and push** | BuildKit builds image and pushes to GHCR on `push: main` | `Fifa_API` |
| **9. CD workflow** | Apply new image SHA, wait for rollout health, fail loudly on regression | `Fifa_API` (deploy job) and this repo (manifests) |
| **10. Autoscaling proven** | Queue 10 jobs, watch the runner pool scale 0 to N to 0. Record. | this repo |
| **11. Shared cache** | MinIO in-cluster, Actions cache action and BuildKit registry cache | this repo plus `Fifa_API` |
| **12. Prod gate** | `myapp-prod` namespace, GitHub Environment with required reviewer, prod stage in CD | this repo plus `Fifa_API` |
| **13. PR previews** | `myapp-pr-{n}` namespaces created on PR open, deployed image, URL posted as comment, cleaned up on close | `Fifa_API` |
| **14. Observability** | kube-prometheus-stack, ServiceMonitors, Grafana dashboards | this repo |
| **15. Security** | NetworkPolicies, ResourceQuota, Trivy scans, gitleaks, OIDC stub | this repo plus `Fifa_API` |
| **16. Polish** | Architecture diagram, runbook, demo recording, README polish | this repo |

## Architectural decisions

These have been locked. Earlier drafts of this doc tracked them as open. They are reflected in the components and phases above.

| # | Decision | Resolution |
|---|---|---|
| 1 | TLS termination at ingress (a) or in-app (b) | (a) at Ingress. App skips in-app TLS when `DISABLE_INLINE_TLS=true`. |
| 2 | Image registry: GHCR or in-cluster Harbor | GHCR. Harbor remains a stretch. |
| 3 | Build tool: Kaniko per-job or rootless BuildKit | rootless BuildKit Deployment. Single in-cluster daemon, jobs talk to it via `buildctl`. |
| 4 | Deploy style: `kubectl apply`, Helm or Argo CD | `kubectl apply -k` (Kustomize). Argo CD is a stretch in Phase 16. |
| 5 | Test framework: Vitest, Jest or Mocha | Vitest. |
| 6 | Cloud target: local-only or also EKS / GKE / AKS | Local-only for v1. Cloud-readiness is a design constraint not a delivery item. |

## Repo layout (planned)

```
KubernetesDeploymentFifaAPI/
.
.. README.md
.. LICENSE
.. .gitignore
.. docs/
   .. ARCHITECTURE.md          this file
   .. RUNBOOK.md               (Phase 16) day-2 ops procedures
   .. DEMO.md                  (Phase 16) how to reproduce the demo end-to-end
.. clusters/
   .. local-k3d/
      .. README.md             how to recreate the cluster, the exact k3d command
.. kubernetes/
   .. namespaces/
   .. app/
      .. myapp-dev/            Deployment, Service, ConfigMap, Secret example, Ingress, RBAC
      .. myapp-prod/
   .. arc/                     ARC values, runner scale set values
   .. buildkit/                rootless BuildKit Deployment, Service, RBAC
   .. cert-manager/
   .. observability/           kube-prometheus-stack values, ServiceMonitors, dashboards
   .. cache/                   MinIO Deployment, Service, ingress for browser
   .. policies/                NetworkPolicies, ResourceQuotas, PodSecurity defaults
.. helm-values/                Helm values files for third-party charts
.. runner-image/               Dockerfile for the custom ARC runner image
.. .github/
   .. workflows/               this repo's CI (lint manifests, kubeconform, helm lint)
                                plus the smoke-test workflow that uses fifa-runners
```

## Drift control

When a pattern is reused 3+ times across phases, promote it to a dedicated rule under `.cursor/rules/` (per the workspace's `self-improvement.mdc`).
