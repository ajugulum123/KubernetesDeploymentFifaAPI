# Observability stack

kube-prometheus-stack (Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics) plus our own ServiceMonitors, PrometheusRule alerts and a Grafana dashboard.

## Apply

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --values KubernetesDeploymentFifaAPI/helm-values/kube-prometheus-stack.yaml

kubectl apply -k KubernetesDeploymentFifaAPI/kubernetes/observability
```

## What gets scraped

| Target | Source | Interval |
|---|---|---|
| ARC controller | `ServiceMonitor arc-controller` in `arc-systems` | 30s |
| Runner pods | `PodMonitor arc-runner-pods` in `arc-runners` | 30s |
| fifa-api | `ServiceMonitor fifa-api` in `myapp-dev` and `myapp-prod` | 30s |
| Cluster default targets | kube-prometheus-stack defaults | 30s |

## Dashboards

The Grafana sidecar discovers any ConfigMap labeled `grafana_dashboard: "1"` in any namespace and loads its JSON dashboards. We ship one:

- `kubernetes/observability/grafana-dashboard-fifa-cicd.yaml`. Title: "FIFA API and CI/CD platform". UID: `fifa-cicd`.

After install, hit `https://grafana.dev.local:8443` (add to /etc/hosts), log in with `admin` / the password set in the Helm values.

## Alerts

`prometheus-rules.yaml` defines:

| Alert | Severity | Trigger |
|---|---|---|
| `ARCQueueDepthHigh` | warning | More than 5 jobs queued for over 5 minutes |
| `ARCControllerDown` | critical | Controller scrape fails for over 2 minutes |
| `FifaAPIErrorRateHigh` | warning | 5xx ratio above 5% over 5 minutes |
| `FifaAPILatencyP99High` | warning | P99 over 1s for 5 minutes |
| `FifaAPIDown` | critical | Scrape target down for over 1 minute |

Wire Alertmanager to a real receiver (Slack, email, Opsgenie) via the `monitoring-kube-prometheus-alertmanager` Secret.

## Verifying after install

```bash
# Are all the components Running?
kubectl get pods -n monitoring

# Did Prometheus pick up the ServiceMonitors?
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
# Then http://localhost:9090/targets

# Is the dashboard loaded?
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana-sc-dashboard | grep fifa-cicd
```
