# BuildKit (rootless, in-cluster)

A single rootless BuildKit daemon runs as a Deployment in the `buildkit` namespace. CI jobs in the `arc-runners` namespace talk to it over gRPC on port `1234` via the `buildkitd.buildkit.svc.cluster.local` Service. No DinD, no `--privileged`, no per-job daemon.

## Why this shape

The two common approaches to building images inside Kubernetes are:

1. **Docker-in-Docker (DinD).** A Docker daemon as a sidecar in each runner pod. Requires `--privileged`. Cache is per-pod, so cold starts on every job. Hard fail.
2. **One in-cluster BuildKit Deployment, jobs are clients.** Daemon is rootless, no privileges, persistent cache via emptyDir or PVC, mTLS optional. Workflows do `buildctl --addr tcp://buildkitd.buildkit.svc:1234 build ...`. This is what we use.

## Apply

```bash
kubectl apply -k kubernetes/buildkit
kubectl wait --for=condition=Available deployment/buildkitd -n buildkit --timeout=120s
kubectl get pods -n buildkit
```

## Smoke test from any pod that has `buildctl`

```bash
BUILDKIT_HOST=tcp://buildkitd.buildkit.svc.cluster.local:1234 \
  buildctl debug workers
```

## Pattern: building and pushing in a workflow

The custom runner image bakes in `buildctl`. Inside a job, build like this:

```bash
echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin

# Hand BuildKit a Docker config so it can authenticate to GHCR for push
mkdir -p ~/.docker
cat > ~/.docker/config.json <<EOF
{
  "auths": {
    "ghcr.io": { "auth": "$(echo -n "${{ github.actor }}:${{ secrets.GITHUB_TOKEN }}" | base64 -w 0)" }
  }
}
EOF

BUILDKIT_HOST=tcp://buildkitd.buildkit.svc.cluster.local:1234 buildctl build \
  --frontend=dockerfile.v0 \
  --local context=. \
  --local dockerfile=. \
  --opt platform=linux/amd64 \
  --output "type=image,name=ghcr.io/${{ github.repository_owner }}/fifa-api:${GITHUB_SHA::12},push=true" \
  --export-cache "type=registry,ref=ghcr.io/${{ github.repository_owner }}/fifa-api:cache,mode=max" \
  --import-cache "type=registry,ref=ghcr.io/${{ github.repository_owner }}/fifa-api:cache"
```

The `cache:` ref pushed alongside the image gives BuildKit a registry-mode cache that survives runner pod tear-downs, which is the whole point of putting the daemon in-cluster.

## Operational notes

- The daemon runs as UID 1000 (`runAsNonRoot: true`).
- The cache lives in `emptyDir: 10Gi`. On node eviction the cache is lost. Replace with a `PersistentVolumeClaim` once an `RWO` StorageClass is in play (`local-path-provisioner` works for k3d).
- Resource limits are conservative. Increase if image builds get OOMKilled on big images.
- The `NetworkPolicy` only allows ingress from the `arc-runners` namespace. Verify with `kubectl describe netpol -n buildkit`.
- For production we add mTLS. See https://github.com/moby/buildkit/blob/master/docs/rootless.md.
