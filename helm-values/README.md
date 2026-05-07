# Helm values

Source-controlled values files for third-party charts. We use Helm only for things published as Helm charts (ARC, kube-prometheus-stack). Our own application is plain manifests under `kubernetes/`.

## Files

| File | Chart | Release name | Namespace |
|---|---|---|---|
| `fifa-runners.yaml` | `gha-runner-scale-set` | `fifa-runners` | `arc-runners` |
| `kube-prometheus-stack.yaml` | `kube-prometheus-stack` (Phase 14) | `monitoring` | `monitoring` |

## Apply or upgrade the runner scale set

```bash
helm upgrade --install fifa-runners \
  --namespace arc-runners --create-namespace \
  --values KubernetesDeploymentFifaAPI/helm-values/fifa-runners.yaml \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

## Inspect the live values

```bash
helm get values fifa-runners -n arc-runners --all
helm get manifest fifa-runners -n arc-runners | head -200
kubectl get autoscalingrunnerset -n arc-runners
```

## Tuning notes

- `minRunners=0` saves resources but every cold start pays the runner-image-pull cost. If your team is hitting the runner pool many times per hour, set `minRunners=1` so one warm runner is always ready.
- `maxRunners=5` is a safety cap. Increase if the queue regularly has more than 5 jobs waiting.
- The runner pod's `resources.limits` cap one job's CPU and memory. A typical `npm test` plus image build for FIFA_API_APP fits in 1 CPU / 2Gi. Bump if jobs OOM or get throttled.
