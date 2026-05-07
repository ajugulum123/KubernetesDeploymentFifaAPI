# Runbook

Day-2 operations and "what to do when X breaks".

## Common operations

### Scale the runner pool

```bash
# Edit minRunners / maxRunners in helm-values/fifa-runners.yaml then:
helm upgrade fifa-runners --namespace arc-runners \
  --values helm-values/fifa-runners.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

### Roll back a bad deploy

```bash
# Find the previous good revision
kubectl rollout history deployment/fifa-api -n myapp-dev

# Roll to it
kubectl rollout undo deployment/fifa-api -n myapp-dev --to-revision=<N>
kubectl rollout status deployment/fifa-api -n myapp-dev
```

### Force a runner image refresh

```bash
helm upgrade fifa-runners --namespace arc-runners \
  --set 'template.spec.containers[0].imagePullPolicy=Always' \
  --values helm-values/fifa-runners.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set

# Recycle existing in-flight pods (if any) so they pick up the new image.
kubectl delete pods -n arc-runners -l actions.github.com/scale-set-name=fifa-runners
```

### Drain the cache

```bash
kubectl exec -n cache deploy/minio -- mc rm --recursive --force minio/actions-cache
```

## Troubleshooting

### Workflow says "no runners available"

Symptoms: a job sits in the queue and pods never appear in `arc-runners`.

Check, in order:

1. The listener pod is up.
   ```bash
   kubectl get pods -n arc-runners | grep listener
   ```
2. The controller is reconciling.
   ```bash
   kubectl logs -n arc-systems deployment/arc-gha-rs-controller --tail=100
   ```
3. The GitHub App permissions are correct (Actions: Read & Write, Administration: Read & Write, Metadata: Read).
4. The queue is actually visible to ARC: in the ARC controller logs you should see lines like `Got message from listener: {"jobsAvailable": ...}`.
5. ResourceQuota exhaustion. `kubectl describe resourcequota -n arc-runners` and look at "Used".

### A 403 from `registration-token`

```
POST .../actions/runners/registration-token: 403 Resource not accessible by integration
```

The GitHub App is missing **Administration: Read & Write** at the repository level. Fix in the App settings page, then accept the new permissions on the install page (`https://github.com/settings/installations/<INSTALLATION_ID>` -> Configure -> the yellow banner). The controller retries every 10s.

### BuildKit refuses connections

```
buildctl: connection refused: tcp://buildkitd.buildkit.svc.cluster.local:1234
```

Check, in order:

1. The Deployment is healthy: `kubectl get pods -n buildkit`.
2. The Service has endpoints: `kubectl get endpoints -n buildkit buildkitd`.
3. The NetworkPolicy in `buildkit/networkpolicy.yaml` allows ingress from `arc-runners`. If you renamed the runner namespace or labels, update the selector.
4. From inside a runner pod: `kubectl exec ... -- nslookup buildkitd.buildkit.svc.cluster.local`.

### MinIO cache writes time out

Usually a NetworkPolicy issue or PVC pressure. Check `kubectl describe pod -n cache -l app.kubernetes.io/name=minio` and look at events.

### Trivy is blocking deploys on a CVE you cannot fix

Two options:

- Accept the risk for a specific CVE: add the CVE to a `.trivyignore` at the repo root.
- Drop the threshold from CRITICAL to "fail only on actively-exploited" by adding `--vuln-type os,library --severity CRITICAL` and an `allowlist`.

### Stale preview namespaces

If the cleanup workflow failed to run, find and remove orphan namespaces:

```bash
kubectl get ns -l app.kubernetes.io/part-of=fifa-cicd-platform | grep myapp-pr-
kubectl delete ns myapp-pr-<N>
```

## Backup and restore

Stateful pieces:

| What | Where | Backup strategy |
|---|---|---|
| Prometheus TSDB | PVC in `monitoring` | `velero backup` or `kubectl cp` of the data dir while paused |
| MinIO data | PVC in `cache` | Disposable. Cache rebuilds from workflows. |
| Grafana dashboards | ConfigMap with `grafana_dashboard: "1"` label | Already in git |
| App data (CSV) | baked into the image | git |

Nothing in the platform should hold customer-impacting data. The pattern is intentional.

## Cluster destroy/recreate

```bash
k3d cluster delete fifa-cicd
k3d cluster create fifa-cicd ...
# then re-run the apply sequence in docs/DEMO.md
```

Total time on a laptop: ~5 minutes.
