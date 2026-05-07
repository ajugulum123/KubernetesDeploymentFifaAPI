# Application manifests

Kubernetes manifests for the FIFA GraphQL API across environments.

## Layout

```
kubernetes/app/
.. myapp-dev/        Deployment, Service, ConfigMap, RBAC, NetworkPolicy, Ingress for dev
.. myapp-prod/       Same shape as dev but locked down further. Requires manual approval.
```

A PR-preview pattern (`myapp-pr-{number}`) is generated dynamically by `.github/workflows/pr-preview.yml` from the dev base. No static manifests for it.

## Apply (manually, the first time)

```bash
# 1. Create the Secret with real values.
kubectl create secret generic fifa-api-secrets \
  --namespace myapp-dev \
  --from-literal=JWT_ACCESS_SECRET="$(openssl rand -hex 64)" \
  --from-literal=JWT_REFRESH_SECRET="$(openssl rand -hex 64)" \
  --from-literal=ADMIN_USERNAME=admin \
  --from-literal=ADMIN_PASSWORD="$(openssl rand -base64 24)"

# 2. Apply everything else.
kubectl apply -k kubernetes/app/myapp-dev

# 3. Set the real image tag (placeholder in deployment.yaml will fail to pull on purpose).
kubectl set image deployment/fifa-api \
  fifa-api=ghcr.io/ajugulum123/fifa-api:<git-sha> \
  --namespace myapp-dev

# 4. Wait for rollout.
kubectl rollout status deployment/fifa-api -n myapp-dev --timeout=120s
```

## Verify

```bash
# Pods Ready and Running
kubectl get pods -n myapp-dev -l app.kubernetes.io/name=fifa-api

# /health via the Service
kubectl run -n myapp-dev curl-test --rm -i --restart=Never --image=curlimages/curl:8.10.1 \
  --command -- curl -fsS http://fifa-api.myapp-dev.svc.cluster.local/health

# /health via the Ingress (after adding fifa-api.dev.local to /etc/hosts)
curl -k https://fifa-api.dev.local:8443/health
```

## RBAC story

- Each app pod runs under the `fifa-api` ServiceAccount in its namespace, with `automountServiceAccountToken: false`. The app does not need Kubernetes API access.
- The deployer SA `app-deployer` lives in `arc-runners` (where the runner pods run). The runner scale set is configured to use this SA via `template.spec.serviceAccountName: app-deployer`.
- Each environment has its own `Role`+`RoleBinding` granting `arc-runners/app-deployer` exactly the verbs+resources it needs in that namespace. No cluster-scope grants. No edit permissions on RBAC, namespaces or PV/PVC.

The RBAC file `myapp-dev/rbac.yaml` is the one to inspect and copy when adding more environments.
