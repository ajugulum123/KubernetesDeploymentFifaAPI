# KubernetesDeploymentFifaAPI

Self-hosted CI/CD platform on Kubernetes for the [FIFA Player Stats GraphQL API](https://github.com/ajugulum123/Fifa_API). Demonstrates a production-grade approach to CI/CD infrastructure: ephemeral runner pods spun up on demand, rootless container builds, RBAC-scoped deploys, queue-depth autoscaling, a shared cache layer, and a Prometheus + Grafana observability stack.

## What this repo contains

This is the **platform repo**. The application source lives in [`ajugulum123/Fifa_API`](https://github.com/ajugulum123/Fifa_API). This repo is responsible for:

- The Kubernetes cluster definition and bootstrap (k3d locally; portable to EKS / GKE / AKS later)
- Actions Runner Controller (ARC) installation and `AutoscalingRunnerSet` configuration
- The container registry, image-build pipeline, and rootless build tooling
- All Kubernetes manifests / Helm charts that deploy the API
- The CD workflow that watches for new application images and rolls them out
- RBAC, NetworkPolicies, and Secrets management
- The observability stack (Prometheus + Grafana + dashboards + ServiceMonitors)
- The shared build cache (MinIO + cache server)

## Architecture at a glance

```
                          push to main
                               |
         +---------------------+----------------------+
         |                                            |
   ajugulum123/Fifa_API                    ajugulum123/KubernetesDeploymentFifaAPI
   (application source)                            (this repo — platform)
         |                                            |
   GitHub Actions workflow                    GitHub Actions workflow
   runs in ARC runner pod                     runs in ARC runner pod
   inside the cluster                         inside the cluster
         |                                            |
   test -> build (Kaniko/BuildKit)            validate manifests
   -> push image to registry                   -> apply / sync to cluster
                                               -> wait for rollout health

   +-----------------------------------------------------------+
   |              Local k3d cluster (fifa-cicd)                 |
   |  ┌────────────────┐  ┌──────────────────┐  ┌────────────┐ |
   |  │ ARC controller │  │ ephemeral runner │  │  fifa-api  │ |
   |  │  + scale set   │  │      pods        │  │ deployment │ |
   |  └────────────────┘  └──────────────────┘  └────────────┘ |
   |  ┌────────────────┐  ┌──────────────────┐  ┌────────────┐ |
   |  │  cert-manager  │  │ Prometheus stack │  │  Traefik   │ |
   |  └────────────────┘  └──────────────────┘  └────────────┘ |
   |  ┌────────────────┐  ┌──────────────────┐                 |
   |  │     MinIO      │  │      registry    │                 |
   |  │ (shared cache) │  │   (GHCR or self) │                 |
   |  └────────────────┘  └──────────────────┘                 |
   +-----------------------------------------------------------+
```

## Status

Scaffolded. Phase 1 (local k3d cluster) is complete. See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full design and phased delivery plan.

## Quick links

| | |
|---|---|
| Application repo | https://github.com/ajugulum123/Fifa_API |
| Architecture doc | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) |
| License | MIT — see [LICENSE](LICENSE) |
