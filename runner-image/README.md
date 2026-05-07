# Custom ARC runner image

The default ARC runner image (`ghcr.io/actions/actions-runner`) is intentionally minimal. Our pipelines need more tooling, so this directory builds a custom runner image and pushes it to GHCR. The `fifa-runners` scale set is configured (in `helm-values/fifa-runners.yaml`) to use this image.

## What is in the image

| Tool | Why |
|---|---|
| `kubectl` | CD steps run `kubectl apply -k`, `kubectl set image`, `kubectl rollout status` |
| `helm` | Third-party chart upgrades from CI when needed |
| `buildctl` | Client for the in-cluster rootless BuildKit. CI builds container images by talking to BuildKit, no Docker daemon required. |
| `docker` (CLI only, no daemon) | Tagging, inspection, occasional `docker login` flows |
| `jq`, `yq` | YAML and JSON manipulation in workflows |
| `node`, `npm` | Vitest, lint and other npm scripts for `Fifa_API` |
| `curl`, `git`, `gettext-base`, `unzip`, `tar`, `xz-utils` | basic plumbing |

We deliberately do not bake in the BuildKit daemon. The daemon lives once in the cluster as a Deployment. Each job mounts a shared cache and connects to it via `BUILDKIT_HOST`. This keeps runner pods unprivileged and small.

## Versions

Pinned via `ARG` in the Dockerfile so the image is reproducible. To bump a tool, edit the `ARG` and rebuild.

| ARG | Default | Where to check for updates |
|---|---|---|
| `ARC_RUNNER_VERSION` | `2.331.0` | https://github.com/actions/runner/releases |
| `KUBECTL_VERSION` | `v1.33.6` | https://kubernetes.io/releases/ |
| `HELM_VERSION` | `v3.16.4` | https://github.com/helm/helm/releases |
| `BUILDKIT_VERSION` | `v0.18.2` | https://github.com/moby/buildkit/releases |
| `NODE_MAJOR` | `20` | https://nodejs.org/en/about/previous-releases |
| `YQ_VERSION` | `v4.44.5` | https://github.com/mikefarah/yq/releases |

The runner version should match (or be one minor behind) the runner version expected by the ARC chart you have installed. The `gha-runner-scale-set-controller` chart 0.14.1 ships runner 2.331.0.

## Building locally

You can build it on your laptop for sanity checks. Push happens via the GitHub Actions workflow.

```bash
docker build \
  --build-arg ARC_RUNNER_VERSION=2.331.0 \
  -t ghcr.io/ajugulum123/fifa-runner:dev \
  KubernetesDeploymentFifaAPI/runner-image
```

## Build and push via GitHub Actions

The workflow `.github/workflows/build-runner-image.yml` builds and pushes on:
- Pushes to `main` that touch `runner-image/`
- Manual dispatch

It tags as:
- `ghcr.io/ajugulum123/fifa-runner:latest`
- `ghcr.io/ajugulum123/fifa-runner:<git-sha>`

## Pointing the scale set at the new image

After the first push completes, edit `helm-values/fifa-runners.yaml` and set:

```yaml
template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/ajugulum123/fifa-runner:latest
```

Then `helm upgrade fifa-runners ...` (the exact command lives in `helm-values/README.md`). New jobs will land on the custom image. Existing in-flight pods finish on the old image.
