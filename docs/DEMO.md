# Demo walkthrough

Reproduce the end-to-end pipeline from a clean checkout in roughly 30 minutes.

## Prerequisites

- Docker Desktop, OrbStack or equivalent. Daemon running.
- `k3d`, `kubectl`, `helm`, `gh` (GitHub CLI), `openssl`. All `brew install`-able.
- A GitHub account with admin on `ajugulum123/Fifa_API` and `ajugulum123/KubernetesDeploymentFifaAPI`.
- A GitHub App with the App ID, Installation ID and private-key PEM already created. See `docs/ARCHITECTURE.md` for the App permissions list.

## 1. Bootstrap the cluster

```bash
k3d cluster create fifa-cicd \
  --servers 1 --agents 3 \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer" \
  --wait

kubectl wait --for=condition=Ready node --all --timeout=120s
```

## 2. Install ARC and the runner scale set

```bash
helm upgrade --install arc \
  --namespace arc-systems --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

kubectl create namespace arc-runners
kubectl create secret generic arc-github-app-secret -n arc-runners \
  --from-literal=github_app_id=<APP_ID> \
  --from-literal=github_app_installation_id=<INSTALLATION_ID> \
  --from-file=github_app_private_key=<path-to-pem>

helm upgrade --install fifa-runners \
  --namespace arc-runners \
  --values helm-values/fifa-runners.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

## 3. Apply platform manifests

```bash
# RBAC
kubectl apply -f kubernetes/rbac/

# BuildKit
kubectl apply -k kubernetes/buildkit

# Cache (create credentials first)
ACCESS=$(openssl rand -hex 16); SECRET=$(openssl rand -hex 32)
kubectl create namespace cache
kubectl create secret generic minio-credentials -n cache \
  --from-literal=accessKey="${ACCESS}" \
  --from-literal=secretKey="${SECRET}" \
  --from-literal=mcHostUrl="http://${ACCESS}:${SECRET}@minio:9000"
kubectl apply -k kubernetes/cache
gh secret set CACHE_ACCESS_KEY --body "${ACCESS}" --repo ajugulum123/Fifa_API
gh secret set CACHE_SECRET_KEY --body "${SECRET}" --repo ajugulum123/Fifa_API

# App namespaces (dev and prod) and Secrets
kubectl apply -k kubernetes/policies
kubectl apply -k kubernetes/app/myapp-dev
kubectl create secret generic fifa-api-secrets -n myapp-dev \
  --from-literal=JWT_ACCESS_SECRET="$(openssl rand -hex 64)" \
  --from-literal=JWT_REFRESH_SECRET="$(openssl rand -hex 64)" \
  --from-literal=ADMIN_USERNAME=admin \
  --from-literal=ADMIN_PASSWORD="$(openssl rand -base64 24)"

kubectl apply -k kubernetes/app/myapp-prod
kubectl create secret generic fifa-api-secrets -n myapp-prod \
  --from-literal=JWT_ACCESS_SECRET="$(openssl rand -hex 64)" \
  --from-literal=JWT_REFRESH_SECRET="$(openssl rand -hex 64)" \
  --from-literal=ADMIN_USERNAME=admin \
  --from-literal=ADMIN_PASSWORD="$(openssl rand -base64 24)"
```

## 4. Install observability

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --values helm-values/kube-prometheus-stack.yaml

kubectl apply -k kubernetes/observability
```

## 5. Configure GitHub Environments (one-time)

In the GitHub UI for `ajugulum123/Fifa_API`:

- Settings, Environments, New environment named `dev`. No protection rules.
- New environment named `production`. Required reviewer: yourself. Branches: `main` only.

## 6. Add /etc/hosts entries

```bash
sudo tee -a /etc/hosts <<EOF
127.0.0.1 fifa-api.dev.local
127.0.0.1 grafana.dev.local
EOF
```

## 7. Build and push the custom runner image (one-time)

The platform repo's `.github/workflows/build-runner-image.yml` does this automatically on push to `main` of `runner-image/**`. To trigger manually:

```bash
gh workflow run build-runner-image.yml --repo ajugulum123/KubernetesDeploymentFifaAPI
gh run watch --repo ajugulum123/KubernetesDeploymentFifaAPI
```

Once it completes, restart the runner scale set so new pods pick up the image:

```bash
helm upgrade fifa-runners \
  --namespace arc-runners \
  --values helm-values/fifa-runners.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

## 8. Drive a deploy

```bash
cd ../Fifa_API
echo "// triggered $(date -u +%FT%TZ)" >> src/server.ts
git add src/server.ts
git commit -m "demo trigger"
git push origin main
```

In one terminal:

```bash
kubectl get pods -n arc-runners -w
```

In another:

```bash
gh run watch --repo ajugulum123/Fifa_API
```

## 9. Verify the deploy

```bash
# /health
curl -k https://fifa-api.dev.local:8443/health

# Issue a GraphQL query
curl -k https://fifa-api.dev.local:8443/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ players(limit: 3) { id name overallRating } }"}'
```

Open Grafana at `https://grafana.dev.local:8443` and watch the request rate climb.

## 10. Approve the prod gate

```bash
RUN_ID=$(gh run list --workflow CD --repo ajugulum123/Fifa_API --limit 1 --json databaseId --jq '.[0].databaseId')
gh run approve "${RUN_ID}" --repo ajugulum123/Fifa_API
```

## 11. Open a PR for the preview environment

```bash
git checkout -b demo-preview
echo "// preview demo" >> src/server.ts
git commit -am "preview demo"
git push -u origin demo-preview
gh pr create --title "preview demo" --body "Will create myapp-pr-{N} namespace" --repo ajugulum123/Fifa_API
```

The `pr-preview.yml` workflow will fire, deploy to a fresh namespace and post a comment with the URL. Closing the PR triggers `pr-preview-cleanup.yml`.

## Recording the demo

Use `asciinema` (or QuickTime + Terminal) to capture the flow:

```bash
asciinema rec -t "fifa-cicd e2e" demo.cast
# run steps 8 and 9
exit
```

Upload `demo.cast` to https://asciinema.org and embed the share URL in the README.
