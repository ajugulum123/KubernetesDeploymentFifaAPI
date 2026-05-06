# Local k3d cluster

The development cluster the platform targets while we're building this thing on a laptop.

## Prerequisites

- Docker (any flavor that exposes a working daemon — Docker Desktop, OrbStack, Colima, Rancher Desktop)
- [k3d](https://k3d.io/) — `brew install k3d`
- [kubectl](https://kubernetes.io/docs/tasks/tools/) — `brew install kubectl`
- [Helm](https://helm.sh/docs/intro/install/) — `brew install helm`

## Create the cluster

```bash
k3d cluster create fifa-cicd \
  --servers 1 \
  --agents 3 \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer" \
  --wait
```

This gives you:

- 1 server (control-plane) + 3 agents (workers) — k3s v1.33.6
- Ports `localhost:8080` → cluster `:80` and `localhost:8443` → cluster `:443`, so any Ingress rule we create can be reached from the host
- Traefik ingress controller installed by default (k3s ships it)
- `local-path-provisioner` as the default StorageClass
- `metrics-server` (HPA-ready)
- kubectl context auto-set to `k3d-fifa-cicd`

## Verify

```bash
kubectl get nodes -o wide          # 4 nodes, all Ready
kubectl get pods -A                # all kube-system pods Running or Completed
kubectl cluster-info               # control plane URL
helm list -A                       # traefik + traefik-crd should be deployed
k3d cluster list                   # fifa-cicd 1/1 servers, 3/3 agents
```

## Lifecycle

```bash
k3d cluster stop fifa-cicd         # pause (Docker containers stopped, no compute used)
k3d cluster start fifa-cicd        # resume
k3d cluster delete fifa-cicd       # nuke everything (irreversible — fast to recreate though)
```

## Known transient issue at boot

`helm-install-traefik` may briefly enter `CrashLoopBackOff` on the first cluster create. Cause: the Traefik chart's `validate-install-crd.yaml` runs before its sibling `helm-install-traefik-crd` Job has finished registering CRDs — known k3s startup race. The Job retries and succeeds on the third attempt. **Don't intervene** unless `kubectl get jobs -n kube-system helm-install-traefik` is still not `Complete` after 90s.

Verify recovery:

```bash
kubectl get jobs -n kube-system    # both helm-install-traefik* jobs Complete
kubectl get crd | grep traefik     # CRDs present (accesscontrolpolicies.hub.traefik.io, etc.)
helm list -A                       # traefik release status: deployed
```
