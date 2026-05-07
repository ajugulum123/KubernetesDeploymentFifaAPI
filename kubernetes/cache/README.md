# Shared cache layer (MinIO + Actions cache)

Self-hosted runners do not get GitHub's hosted Actions cache for free. We run a single-node MinIO in-cluster as an S3-compatible store, and workflows use [`tespkg/actions-cache`](https://github.com/tespkg/actions-cache) to read and write cache entries against it.

The same MinIO is suitable for BuildKit registry-mode caching too, though our default in this repo is to keep BuildKit's cache in GHCR (`ghcr.io/<owner>/<image>:cache`).

## Apply

```bash
# 1. Create the credentials Secret (one-time)
ACCESS=$(openssl rand -hex 16)
SECRET=$(openssl rand -hex 32)
kubectl create namespace cache --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic minio-credentials \
  --namespace cache \
  --from-literal=accessKey="${ACCESS}" \
  --from-literal=secretKey="${SECRET}" \
  --from-literal=mcHostUrl="http://${ACCESS}:${SECRET}@minio:9000"

# 2. Apply the rest
kubectl apply -k kubernetes/cache

# 3. Wait for the bucket-init Job
kubectl wait --for=condition=Complete job/minio-bucket-init -n cache --timeout=120s
```

## Upload the keys to GitHub as workflow secrets

```bash
# Print so you can copy them into GitHub repository secrets named
# CACHE_ACCESS_KEY and CACHE_SECRET_KEY.
kubectl get secret minio-credentials -n cache -o jsonpath='{.data.accessKey}' | base64 -d ; echo
kubectl get secret minio-credentials -n cache -o jsonpath='{.data.secretKey}' | base64 -d ; echo
```

Or via `gh secret set`:

```bash
gh secret set CACHE_ACCESS_KEY --body "$(kubectl get secret minio-credentials -n cache -o jsonpath='{.data.accessKey}' | base64 -d)" --repo ajugulum123/Fifa_API
gh secret set CACHE_SECRET_KEY --body "$(kubectl get secret minio-credentials -n cache -o jsonpath='{.data.secretKey}' | base64 -d)" --repo ajugulum123/Fifa_API
```

## Use it in a workflow

Replace `actions/cache@v4` with `tespkg/actions-cache@v1` in any cache step:

```yaml
- uses: tespkg/actions-cache@v1
  with:
    endpoint: minio.cache.svc.cluster.local
    port: 9000
    accessKey: ${{ secrets.CACHE_ACCESS_KEY }}
    secretKey: ${{ secrets.CACHE_SECRET_KEY }}
    bucket: actions-cache
    insecure: true
    use-fallback: false
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-
    path: ~/.npm
```

The `actions/cache` and `tespkg/actions-cache` APIs are nearly drop-in compatible. `endpoint` plus `port` plus `bucket` plus credentials are the only extra fields.

## Inspect cache contents

```bash
# Port-forward the console
kubectl port-forward -n cache svc/minio 9001:9001
# then open http://localhost:9001 (login with the access/secret key)
```

## Operational notes

- Single-node, single-PVC. No replication. This is a CI cache.
- 10Gi PVC by default. Bump in `minio.yaml` PVC if jobs hit the cap.
- Cache entries are scoped per workflow's `key`. Cleanup is the workflow author's responsibility today. Add a CronJob with `mc rm --recursive --older-than 7d` if cache grows uncontrollably.
- Replace with a managed S3 (AWS, GCS in S3 mode, R2) when moving off-laptop.
